---
title: jupyter notebook使用
date: 2018-09-15 23:33:18
categories:
- python
tags:
- jupyter
- 工具
copyright:
top:
type:
---
# jupyter notebook

Jupyter notebook, 前身是 IPython notebook, 它是一个非常灵活的工具，有助于帮助你构建很多可读的分析，你可以在里面同时保留代码，图片，评论，公式和绘制的图像。
这篇文章主要是记录下jupyter notebook使用过程中一些常用的内容和技巧，包括一些magic使用

## 快捷键

windows上，**Ctrl + Shift + P**可以调出快捷键的帮助信息，所有的快捷键都能够看到。掌握快捷键能够帮助我们更好的使用jupyter notebook

## Jupyter Magic Commands

1. %matplotlib inline

如果想在jupyter中画图，可以添加如下代码，则图像就能够进行显示

```python
%matplotlib inline
```

2\. Timing

对于计时有两个十分有用的魔法命令：%%time 和 %timeit. 如果你有些代码运行地十分缓慢，而你想确定是否问题出在这里，这两个命令将会非常方便。

**%%time** 将会给出cell的代码运行一次所花费的时间。

```python
%%time
import time
for _ in range(1000):
    time.sleep(0.01)# sleep for 0.01 seconds
```

**%timeit** 使用Python的timeit模块，它将会执行一个语句100，000次(默认情况下)，然后给出运行最快3次的平均值。

```python
import numpy
%timeit numpy.random.normal(size=100)
```

## 运行文件
