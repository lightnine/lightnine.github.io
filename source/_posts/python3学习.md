---
title: python3学习
date: 2018-07-10 11:48:04
categories:
tags:
copyright:
top:
type:
---

# 模块

在python中,一个py文件就是一个模块.而为了避免模块名称冲突,Python又引入了按目录来组织模块的方法,成为Package.引入了包后,对应的模块名前面就添加了包名.类似于Java中的Package.

\_\_init\_\_.py:在包目录下,必须有\_\_init\_\_.py这个文件,如果没有这个文件,那么python会将此文件作为一个普通目录,而不在是一个package.\_\_init\_\_.py可以为空,也可以有代码(一般都是一些import语句).\_\_init\_\_.py本身是一个模块,其模块名为对应的package名称.

# Python示例程序

```python hello.py
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

' a test module '

__author__ = 'Michael Liao'

import sys

def test():
    args = sys.argv
    if len(args)==1:
        print('Hello, world!')
    elif len(args)==2:
        print('Hello, %s!' % args[1])
    else:
        print('Too many arguments!')

if __name__=='__main__':
    test()
```

程序说明:程序开始的一,二行是标准注释.第一行注释说明这个文件可以直接在Unix/Linux/Max上运行.第二行表示文件本身使用utf-8编码.第四行是模块的文档注释,第六行是作者信息.
`if __name__ == '__main__':`:当我们在命令行运行模块时,python解释器会将一个特殊的变量`__name__`设置为`__main__`.但是如果在其他模块中导入此模块,python解释器并不是设置`__name__`变量.

# python中的作用域

正常的变量或函数是公开的,可以直接引用如`abc`,`xyz`等.
类似`__xxx__`的变量时特殊变量,也可以直接引用,但是有特殊用途.如`__author__`,`__name__`.而模块的文档注释可以用`__doc__`来访问
类似于`_x`,`__x`这样定义的变量或函数是非公开的,类似Java的private.但是Python中没有方法可以限制private的访问.所以这样定义的变量或者函数不应该被直接引用.