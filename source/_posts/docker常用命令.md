---
title: docker常用命令
date: 2018-07-31 16:53:41
categories:
- docker
tags:
- docker
copyright:
top:
type:
comments: true
---
# docker基本命令

## 查看docker版本

`docker version`

## 查看docker信息

`docker info`

## 启动docker 服务

```bash
sudo service docker start
或者
sudo systemctl start docker
```

## 列出本机的所有image文件

`docker image ls`

## 删除image文件

`docker image rm [imageName]`

## 拉取image文件

`docker image pull library/hello-world`
library/hello-world是 image 文件在仓库里面的位置，其中library是 image 文件所在的组，hello-world是 image 文件的名字.因为Docker官方提供的镜像都在library中,所以可以省略library,即使用下面的命令
`docker image pull hello-world`

## 运行image 文件

`docker container run hello-world`
run命令每运行一次,就会新建一个容器.docker container run命令具有自动抓取 image 文件的功能。如果发现本地没有指定的 image 文件，就会从仓库自动抓取。因此，前面的docker image pull命令并不是必需的步骤

## 终止容器

`docker container kill [containID]`

> image 文件生成的容器实例，本身也是一个文件，称为容器文件。也就是说，一旦容器生成，就会同时存在两个文件： image 文件和容器文件。而且关闭容器并不会删除容器文件，只是容器停止运行而已。

## 列出本机正在运行的容器

`docker container ls`

## 列出本机所有容器，包括终止运行的容器

`docker container ls --all`

## 删除容器文件

`docker container rm [containerID]`

## 创建image 文件

```bash
docker image build -t koa-demo .
或者
docker image build -t koa-demo:0.0.1 .
```

**-t** 用来指定image 文件的名字,后面可以用冒号指定标签.如果不指定,默认的标签就是**latest**. 最后的点表示Dockerfile文件所在的路径,一个点表示当前路径

## 从image文件生成容器

```bash
docker container run -p 8000:3000 -it koa-demo /bin/bash
或者
docker container run -p 8000:3000 -it koa-demo:0.0.1 /bin/bash
```

参数含义:

```yml
-p参数: 容器的3000端口映射到本机的8000端口.
-it参数: 容器的Shell映射到当前的Shell,然后你在本机窗口输入的命令,就会传入容器.
koa-demo:0.0.1: image文件的名字,默认标签是latest.
/bin/bash: 容器启动以后,内部第一个执行的命令.这里启动Bash,保证用户可以使用Shell
```

`docker container run --rm -p 8000:3000 -it koa-demo /bin/bash`
--rm参数:在容器终止运行后自动删除容器文件

## 其他有用的命令

1. docker container start

前面的docker container run命令是新建容器，每运行一次，就会新建一个容器。同样的命令运行两次，就会生成两个一模一样的容器文件。如果希望重复使用容器，就要使用docker container start命令，它用来启动已经生成、已经停止运行的容器文件。
`docker container start [containerID]`

2\. docker container stop
前面的docker container kill命令终止容器运行，相当于向容器里面的主进程发出 SIGKILL 信号。而docker container stop命令也是用来终止容器运行，相当于向容器里面的主进程发出 SIGTERM 信号，然后过一段时间再发出 SIGKILL 信号.
`docker container stop [containerID]`
这两个信号的差别是，应用程序收到 SIGTERM 信号以后，可以自行进行收尾清理工作，但也可以不理会这个信号。如果收到 SIGKILL 信号，就会强行立即终止，那些正在进行中的操作会全部丢失。

3\. docker container logs
docker container logs命令用来查看 docker 容器的输出，即容器里面 Shell 的标准输出。如果docker run命令运行容器的时候，没有使用-it参数，就要用这个命令查看输出。
`docker container logs [containerID]`

4\. docker container exec
docker container exec命令用于进入一个正在运行的 docker 容器。如果docker run命令运行容器的时候，没有使用-it参数，就要用这个命令进入容器。一旦进入了容器，就可以在容器的 Shell 执行命令了。
`docker container exec -it [containerID] /bin/bash`

5\. docker container cp
docker container cp命令用于从正在运行的 Docker 容器里面，将文件拷贝到本机。下面是拷贝到当前目录的写法。
`docker container cp [containID]:[/path/to/file] .`

参考文章:[Docker 入门教程-阮一峰](http://www.ruanyifeng.com/blog/2018/02/docker-tutorial.html)