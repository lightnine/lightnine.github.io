---
title: python3学习
date: 2018-07-10 11:48:04
categories:
tags:
 - python
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
类似于`_x`,`__x`这样定义的变量或函数是非公开的,类似Java的private.

# python模块搜索路径

当我们引入模块时,即使用import导入模块时,python解释器从搜索路径中查找对应的模块.如果找不到,则会报错.
默认情况下,Python解释器会搜索当前目录、所有已安装的内置模块和第三方模块，搜索路径存放在sys模块的path变量中.下面代码可以查看搜索路径内容

```bash
>>> import sys
>>> sys.path
```

## 修改搜索路径

可以使用两种方式修改搜索路径

1. 直接修改`sys.path`
    ```bash
    >>> import sys
    >>> sys.path.append('待添加路径')
    ```
    但是这种方式只在程序运行时有效,程序运行结束后失效
2. 设置`PATHONPATH`
    直接将待添加的路径加入`PATHONPATH`环境变量中即可

# 类与对象

在python类中,`__init__`方法就相当于Java中类的构造函数,**self**表示实例自身.
如果类中的变量定义前加了两个下划线,那么此变量就成为一个私有变量

```python
class Student(object):
    def __init__(self, name, score):
        self.__name = name
        self.score = score
a = Student('1', 2)
print(a.__name)
print(a.score)
```

打印`__name`时,将会报错,而可以打印score的内容.

## 鸭子类型

>注明:个人解决因为无法限制方法入参的类型,那么传递进去什么都可以

```python
class Animal(object):
    def run(self):
        print('Animal is running...')
def run_twice(animal):
    animal.run()
    animal.run()
class Timer(object):
    def run(self):
        print('Start...')
run_twice(Timer())
```

在Java中,如果定义`run_twice`方法,必须定义函数的形参类型,如果指定了只能接受Animal类型,则只能传递Animal类型或其子类.但是在python不是这样,只要传递进的类型中存在`run`方法即可.不过python中不存在限制入参是什么类型.
> 这就是动态语言的“鸭子类型”，它并不要求严格的继承体系，一个对象只要“看起来像鸭子，走起路来像鸭子”，那它就可以被看做是鸭子。


