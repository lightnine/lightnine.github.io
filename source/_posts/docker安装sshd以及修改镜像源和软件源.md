---
title: docker安装sshd以及修改镜像源和软件源
date: 2018-12-20 16:26:08
categories:
- docker
tags:
- 镜像源
- 软件源
copyright:
top:
type:
---
# 总览

因为最近公司需要在容器中跑深度学习任务，所以需要tensorflow的镜像。并且要求在要通过ssh进行容器的通信。这篇文章主要介绍如何在docker容器中安装软件，如何修改docker镜像源以及修改容器内部软件源地址。
宿主机环境：

```bash
uname -r : 3.10.0-862.11.6.el7.x86_64
cat /etc/redhat-release : CentOS Linux release 7.5.1804 (Core)
docker version : 18.06.0-ce
```

镜像主要选择的是官方的tensorflow:1.12.0-py3版本,镜像里面使用的系统环境下面会介绍

## 修改docker镜像源

[!参考](https://www.docker-cn.com/registry-mirror)
由于docker 官方的镜像源在国外，导致下载速度很慢。所以第一步是修改docker镜像源地址。我这里采用永久修改docker镜像源地址。打开`/etc/docker/daemon.json`（如果没有此文件，则新建）,添加如下内容：

```yaml
{
  "registry-mirrors": ["https://registry.docker-cn.com"]
}
```

然后运行`docker pull registry.docker-cn.com/tensorflow/tensorflow:1.12.0-py3`
即可以拉取镜像，而且可以看到速度提升很快

## 启动镜像

我们可以使用`docker images`查看本机有的镜像。针对上面拉取的tensorflow:1.12.0-py3镜像，我们执行下面命令启动

```bash
docker run -it [image_name] /bin/bash
```

此时我们应该是在启动的容器中。我们可以查看容器具体的系统信息。因为tensorflow基础镜像是ubuntu，所以这里运行以下命令,因为容器中一般不会有vim命令，所以采用cat命令参看系统信息。注意下面的命令都是在容器中键入

```bash
uname -r
cat /etc/issue
```

查看得到系统信息为:Ubuntu 16.04.5 LTS

## 修改容器中软件源地址

由于容器中软件源默认地址是国外的，这里换成国内的。

```bash
cd /etc/apt
cp sources.list sources.list.bak
```

这里使用阿里的镜像源，将sources.list里面的内容修改为如下内容:

```yaml
# 默认注释了源码镜像以提高 apt update 速度，如有需要可自行取消注释
deb http://mirrors.aliyun.com/ubuntu/ xenial main restricted universe multiverse
# deb-src http://mirrors.aliyun.com/ubuntu/ xenial main main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates main restricted universe multiverse
# deb-src http://mirrors.aliyun.com/ubuntu/ xenial-updates main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-backports main restricted universe multiverse
# deb-src http://mirrors.aliyun.com/ubuntu/ xenial-backports main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-security main restricted universe multiverse
# deb-src http://mirrors.aliyun.com/ubuntu/ xenial-security main restricted universe multiverse

# 预发布软件源，不建议启用
# deb http://mirrors.aliyun.com/ubuntu/ xenial-proposed main restricted universe multiverse
# deb-src http://mirrors.aliyun.com/ubuntu/ xenial-proposed main restricted universe multiverse
```

## 容器中安装软件

```bash
passwd
apt-get update
apt-get install vim
apt-get install openssh-sshd
```

修改root密码以及安装完上面vim和openssh-sshd两个软件后，修改openssh-sshd配置文件内容。打开`/etc/ssh/sshd_config`将`PermitRootLogin `后面添加yes，即允许以root用户登录。后面启动sshd命令可能会报错，具体问题可以自己上网上搜索

## 生成镜像

退出镜像，将容器提交为镜像

```bash
docker commit [containerID] [image_name]
```

执行`docker images`可以看到生成的镜像。启动此镜像并开启sshd服务命令如下

```bash
docker run -d  -p 50001:22 [images_name] /usr/sbin/sshd -D
其中 -d表示已后台方式启动容器，-p挂载端口 ，/usr/sbin/sshd -D 是启动命令
```