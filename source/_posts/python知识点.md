---
title: python知识点
date: 2018-09-07 22:12:22
categories:
- python
tags:
copyright:
top:
type:
---
> 都是关于python3的内容

# Python知识点

这里主要是记录下我平常编程中用到，但是不太清楚地知识点，仅仅作为一个记录使用

## python基本类型

1. Number(数字)
2. String(字符串)
3. List(列表)
4. Tuple(元组)
5. Set(集合)
6. Dictionary(字典)

不可变数据：Number（数字）、String（字符串）、Tuple（元组）；
可变数据：List（列表）、Dictionary（字典）、Set（集合）。
使用type()可以查看变量的类型，用isinstance()也可以查看。区别：

- type()不会认为子类是一种父类类型
- isinstance()认为子类是一种父类类型

如下代码所示：

```python
class A:
    pass

class B(A):
    pass

isinstance(A(), A)  # returns True
type(A()) == A      # returns True
isinstance(B(), A)    # returns True
type(B()) == A        # returns False
```

### Number

Python3中Number包括：int,float,bool,complex.注意只有一种整数类型int，表示长整型。

## numpy相关

### 维度问题

注意下面的col_r1和col_r2的输出以及对应维度

```python
a = np.array([[1,2,3,4], [5,6,7,8], [9,10,11,12]])
print(a)
col_r1 = a[:, 1]
col_r2 = a[:, 1:2]
print(col_r1)
print(col_r1.shape)
print(col_r2)
print(col_r2.shape)
```

### numpy数组访问

注意这里使用数组索引来访问numpy数组。这里可以看成[[[0, 1, 2], [0, 1, 0]]]中(0,0),(1,1),(2,0)看成三对
```python
import numpy as np
a = np.array([[1,2], [3, 4], [5, 6]])
print(a[[0, 1, 2], [0, 1, 0]])  # Prints "[1 4 5]"
```

1. numpy中有mat和array，都可以用来表示多维数据。如果一个