---
title: pip以及conda镜像源修改及命令使用
date: 2019-03-25 16:57:15
categories:
- python
tags:
- pip
- conda
copyright:
top:
type:
---

# Python包的安装

在国内环境下，因为网络原因，所以Python下很多包安装不了或者安装的速度很慢。这里主要介绍下如何修改conda以及pip对应的镜像源。

## pip

pip主要是用来管理python包的工具，类似于Maven工具。

### 临时修改pip安装源

比如我们要安装gevent包，我们可以输入一下命令

```bash
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple gevent
```

这样就会从清华这边的镜像去安装gevent库.其中`-i`参数指定了使用清华的pip源

有时候可能需要添加受信源，命令如下：

```bash
pip install packagename -i http://pypi.douban.com/simple --trusted-host pypi.douban.com
```

其中`--trusted-host` 参数是指设置为受信源，否则在安全性较高的连接下是连接不上的

### 永久修改

在用户根目录(~，而非系统根目录 / )下添加配置~/.pip/pip.conf目录添加可信源，如果目录文件不存在，可直接创建。写入如下内容：

```yaml
[global]
index-url=http://pypi.douban.com/simple
trusted-host = pypi.douban.com 
```

这里添加的是豆瓣源，也可以添加清华源

## conda使用及源修改

conda是Anaconda中用来安装python包的工具。在Anaconda中将镜像分为两类，一类是官方的python包，放在anaconda中；另一类是第三方的python包，放在conda-forge中。

采用conda 安装python包时，可以使用以下命令：

```bash
conda install -c https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/linux-64 joblib
```

其中参数`-c`指定了镜像源的通道，这里实在anaconda官方中安装joblib

或者

```bash
conda install -c https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/conda-forge/linux-64 jieba
```

这里是在conda-forge中安装jieba第三方的Python包
