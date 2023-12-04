---
title: java-zero-copy
date: 2023-12-04 11:17:19
categories:
- linux
tags:
- java
- linux
- zero-copy
copyright:
top:
type:
---

> 本篇内容主要翻译自[Efficient data transfer through zero copy](https://developer.ibm.com/articles/j-zerocopy/)，包括有些自己的思考
>

# java zero copy
很多网页应用有大量的静态内容，针对这些静态内容主要的处理逻辑就是从磁盘读取数据，然后将数据写入到响应的socket中。这项工作应该需要较少的CPU资源。但是有时候并不是。内核从磁盘读取数据，然后将数据拷贝到应用中。之后应用将数据写入到内核，然后推送到socket中。实际上，应用程序在这里扮演了一个无效率的中间层，既将数据从磁盘写入到socket。
每一次当数据扩容user-kernel边界时，数据都会被拷贝，而这会消耗cpu cycles以及内存带宽。幸运的是，我们可以采用zero copy技术来避免内核和应用程序之间的数据拷贝。应用程序使用zero copy技术来请求内核直接将数据从磁盘文件拷贝到socket中，而不需要经过应用程序。zero copy技术能够极大的提升应用程序性能并且减少内核空间和用户空间之间的切换。
Java中使用`java.nio.channels.FileChannel`中的`transferTo()`方法在linux和Unix系统重实现zero copy。使用`transferTo()`方法能够直接将字节数据从一个channel写入到另一个channel中，而数据不需要经过应用程序。本篇文章首先展示使用传统的拷贝方法消耗的资源，然后展示使用zero copy获得的性能提升。

## 数据传输:传统方法
想想一个简单的场景，从一个文件读取数据，然后将数据通过网络写入到另一个程序中。核心操作如下所示。
```java
File.read(fileDesc, buf, len);
Socket.send(socket, buf, len);
```
虽然看起来很简单，但是这个操作需要再内核空间和用户空间4次上下文切换，以及4次数据拷贝。下图展示了具体过程
{% asset_img figure1.gif 传统的数据拷贝过程 %}
下图展示了上下文切换过程
{% asset_img figure2.gif 上下文切换 %}

主要的步骤如下：
1. `read()`方法从user mode 转换到 kernel mode。在read内部是发一起了一次系统调用`sys_read()`从文件读取数据。第一次数据拷贝是由DMA引擎来执行，DMA从磁盘读取文件，然后将数据保存到内核缓冲区中。
2. 请求的数据被从内核缓冲区拷贝到用户缓冲区，`read()`方法返回。这引起了第二次的上下文切换。现在数据是在用户空间中的缓冲区中。
3. `send()`方法调用引起用户空间到内核空间的切换。这次会将数据从用户空间拷贝到内核缓冲区。这一次数据是放置到另外一个内核缓冲区中，与目的socket相关联。
4. `send()`方法返回，又引起了一次上下文切换。DMA将数据从内核缓冲区拷贝到网卡缓冲区中，这是第四次数据拷贝。

我们为什么不直接讲数据拷贝到用户空间，而要经过内核空间呢？这是因为操作系统引入内核缓冲区是为了提升性能。操作系统读取数据都会预读取一些数据，这样在应用程序读取额外的数据时，可以不用发起系统调用，直接从内核缓冲区获取即可。而写入过程，可以完全实现为异步过程。既应用程序只需要将数据写入到内核缓冲区中，而不是写入到磁盘中。
但是，设想当前应用程序需要处理的数据要远远大于内核空间缓冲区的大小。而此时，数据需要在磁盘，内核缓冲区，用户空间中来回拷贝。这会严重影响性能。
Zero copy技术是解决这个问题的方法。

## 数据传输：zero copy方法
如果你仔细检查上面的过程，你会发现第二次和第三次数据拷贝可以省略。应用程序针对这些数据什么也不做。因此数据可以被直接从内核缓冲区拷贝到socket buffer中。`transferTo()`方法可以完成这个操作。下面展示了此方法
```java
public void transferTo(long position, long count, WritableByteChannel target);
```
`transferTo()`方法将数据从文件channel拷贝到target channel中。这个方法依赖底层操作系统对于zero copy的支持。在UNIX或linux中，使用的是`sendfile()`系统调用。如下所示：
```c
#include <sys/socket.h>
ssize_t sendfile(int out_fd, int in_fd, off_t *offset, size_t count);
```
下图展示了`transferTo()`方法执行过程
{% asset_img figure3.gif transferTo过程 %}

下图展示了具体的上下文切换
{% asset_img figure4.gif 上下文切换 %}

`transferTo()`方法执行过程如下：
1. DMA引擎将文件内容拷贝到内核缓冲区。然后数据从内核缓冲区拷贝到socket buffer中。(涉及到两次数据拷贝)
2. 第三次数据拷贝发生在DMA引擎将数据从socket buffer拷贝到网卡中。

我们将上下文切换从4次降低到2次，数据拷贝从4次降低到3次(其中仅有一个数据拷贝需要CPU参与)。但是这还没有达到zero copy的目的。如果底层网卡支持gather操作，那么我们可以减少内核空间中的数据重复。在linux kernel2.4及以后版本中，socket buffer已经支持了这个操作。这个方法不仅仅减少了上下文切换并且也消除了CPU参与的数据拷贝。具体如下：
1. `transferTo()`方法将文件内容拷贝到内核缓冲区，由DMA引擎执行
2. 不需要将数据拷贝进socket buffer中，仅仅将数据的位置以及数据长度添加到socket buffer中。DMA引擎直接将kernel buffer中的数据拷贝到网卡中。
下图展示了包含gather操作的`transferTo()`
{% asset_img figure5.gif gather操作的transferTo方法 %}

## 性能比较
使用java实现了文件传输，同时采用传统的IO和nio来实现。完整代码[参考](https://github.com/lightnine/j-zerocopy).性能比较如下：

| file size | traditional(ms) | nio(ms) |
|-----------|-----------------|---------|
| 12MB      | 50              | 18      |
| 221MB     | 690             | 314     |
| 2.5G      | 15496           | 2610    |

评测环境：
1. java : openjdk version "1.8.0_382", OpenJDK Runtime Environment (build 1.8.0_382-b05), OpenJDK 64-Bit Server VM (build 25.382-b05, mixed mode)
2. linux: CentOS Linux release 7.6 (Final)
3. kernel: 4.14.0_1-0-0-51

