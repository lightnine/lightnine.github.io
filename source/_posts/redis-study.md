---
title: redis study
date: 2024-04-01 19:33:09
categories: redis
tags: redis
copyright:
top:
type:
---

## redis 数据结构
### 简单动态字符串
redis中字符串是自己定义的结构，名为SDS。不是用的c语言的字符串。
SDS的定义如下：
```c
struct sdshdr {
    // 记录buf数组中已使用字节的数量
    int len;
    // 记录buf数组中未使用字节的数量
    int free;
    // 字节数组，用于存储字符串;其中最后一位保存'\0'字符
    char buf[];
}
```
为什么要自己实现，而不是复用c中对于字符串的定义？
- 获取字符串长度，在O(1)复杂度获取
- 杜绝缓冲区溢出。SDS在进行修改时，会检查SDS空间是否满足要求。若不满足，则会进行扩容。
- 二进制安全，c中的字符串必须符合某种编码格式(比如ASCII)，并且除了字符串的末尾之外，字符串中不能包含空字符。而sds使用len记录字符串长度，所以是二进制安全的。

SDS的扩容机制：
- 空间预分配（若对sds进行修改，先将len设置为需要的空间大小）
  - 若sds中len小于1MB，则将free设置为跟len一样的大小
  - 若sds中len大于1MB，则将free设置为1MB
- 惰性空间释放
  - 若释放sds中内容，则free中进行增加。实际占用的空间不释放；当然也提供了相应的api，在需要时，真正释放sds未使用空间。
  

### 链表
redis中链表结构
链表中节点定义
```c
typedef struct listNode {
    // 前置节点
    struct listNode *prev;
    // 后置节点
    struct listNode *next;
    // 节点值
    void *value;
} listNode;
```
链表定义
```c
typedef struct list {
    // 表头节点
    listNode *head;
    // 表尾节点
    listNode *tail;
    // 链表包含的节点数量
    unsigned long len;
    // 节点值复制函数
    void *(*dup) (void *ptr);
    // 节点值释放函数
    void (*free) (void *ptr);
    // 节点值对比函数
    int (*match) (void *ptr, void *key);
} list;
```
注：链表节点使用 void*指针保存节点值，并且通过list中的dup、free、match三个属性为节点值设置类型特定函数，所以链表可以保存各种不同类型的值。

### 字典
### 跳跃表
### 整数集合
### 压缩列表
### 对象
