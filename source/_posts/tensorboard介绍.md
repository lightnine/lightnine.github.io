---
title: tensorboard介绍
date: 2018-12-26 11:23:15
categories:
- tensorflow
tags:
- tensorboard
copyright:
top:
type:
---
最近因为工作需要，看了下tensorboard的使用方式，这里做了简单的学习记录。使用的tensorflow版本是**1.10.0**,tensorboard的版本也是一样。

# tensorboard介绍

tensorboard主要是为了查看tensorflow程序，将程序进行图示化，可以用来查看程序的运行情况，也可以用来debug程序。

## tensorflow写tensorboard日志

在程序中主要使用`tf.summary.FileWriter`来将需要记录的事件日志记录到硬盘中。这个函数主要由三个参数需要注意的，如果想要查看详细的内容，可以去源码中查找具体的使用方式。

```yaml
logdir:指定日志写入的具体目录
graph:指定记录的tensorflow图
filename_suffix:事件日志文件的后缀
```

## tensorboard启动

tensorboard的启动是在命令行中，输入`tensorboard --logdir=/path/to/log`命令启动，然后打开ip+port在浏览器中进行查看。查看tensorboard的参数信息，可以运行`tensorboard --help`来进行查看。

```yaml
--inspect: 用来查看事件日志信息，并不启动tensorboard
--logdir：用来指定事件日志的存储目录
```

tensorboard主要是扫描logdir指定目录下的内容，扫描文件名带有*.tfevents.*的文件进行展示。如果logdir下有多个目录，则在浏览器中可以分别进行浏览。如果有多个相同的文件，则会显示时间戳距离最近的文件。文件名中带有时间戳。例如：**events.out.tfevents.1545795299.DESKTOP-123**