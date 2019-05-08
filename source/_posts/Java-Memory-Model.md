---
title: Java Memory Model
date: 2019-05-06 18:58:33
categories:
- java
tags:
- 并发
- 内存模型
copyright:
top:
type:

---

Java内存模型规定了Java虚拟机如何跟计算机的内存协同工作。因为Java虚拟机模拟了计算机，所以自然Java虚拟机包括内存模型。

正确理解Java内存模型对于编写正确的并发程序非常重要。Java内存模型规定了线程何时以及怎么读取其他线程写的值，还有就是在获取共享变量时如何进行同步操作。

最初的Java内存模型是有缺陷的，因此在Java5中进行了修改，并且这个版本的Java内存模型一直到Java8都在使用。

# Java内存模型

JVM中的内存模型将内存分为线程栈内存和堆内存。下面的图从逻辑上展示了Java内存模型:
{% asset_img java-memory-model-1.png java-memory-model %}

每个在JVM中运行的线程都有它自己栈空间。线程的栈空间中包含了线程调用方法执行到那一刻的数据。随着线程执行它的代码，调用栈也随之改变。

线程的栈空间同样也包含每个执行的方法的局部变量。线程只能获取它自己的栈空间，其中包含的局部变量对于其他线程是不可见的。即使两个线程执行相同的代码，这两个线程也是在各自的线程栈空间中创建各自的局部变量。

所有的原型类型(boolean,byte,short,char,int,long,float,double)的局部变量都是存储在线程的栈空间中。一个线程可能传递一个原型变量的副本给另一个线程，但是另一个线程并不能共享这个原型变量。

不管哪个线程创建了对象，这些对象都是存储在堆空间中。这也包括原型类型的包装器类型(e.g. Byte, Integer)。不管对象是被分配给局部变量还是作为另一个对象的成员变量，这个对象都是存储在堆空间中。

下面的图中说明了调用栈和局部变量存储在线程的栈空间中，而对象存储在堆空间中:

{% asset_img java-memory-model-2.png java-memory-model %}

如果局部变量是原型类型，那么这个变量在线程的栈空间中。

如果局部变量是一个指向对象的引用类型，那么这个引用是在线程的栈空间，但是对象本身是在堆空间中。

如果一个对象包含方法，同时这些方法包含局部变量。那么这些局部变量(原型类型或者引用)是保存在线程栈空间中，即使这些方法所在的对象是存储在堆空间中。

一个对象的成员变量是跟对象一起存储在堆内存中，不管这个成员变量是原型还是指向一个对象的引用。

类的静态变量也是跟类的定义一起存在堆内存中。

堆空间中对象能够被所有拥有指向此对象的引用的线程访问到。当一个线程能够访问一个对象时，那么这个线程也能够访问此对象的成员变量(这里要看这个对象的封装性)。如果两个线程同时调用了同一个对象的一个方法，那么这两个线程将能够访问这个对象的成员变量，但是每个线程都会获得局部变量的一份拷贝。

下面的图说明了上面描述的内容：

{% asset_img java-memory-model-3.png java-memory-model %}

在上图中，两个线程有一系列的局部变量。其中一个局部变量(Local Variable 2)指向了堆内存上的共享对象(Object3)。这两个线程分别拥有一个指向同一个对像的引用，这两个引用是不同的。这些引用是局部变量，所以存储在各自线程的栈空间中。而这两个不同的引用指向了堆上的同一个对象。

注意到共享对象(Object3)有两个引用指向了Object2和Object4,如图中箭头所示。通过Object3中的成员变量引用，这两个线程能够获取Object2和Object4。

> The diagram also shows a local variable which point to two different objects on the heap. In this case the references point to two different objects (Object 1 and Object 5), not the same object. In theory both threads could access both Object 1 and Object 5, if both threads had references to both objects. But in the diagram above each thread only has a reference to one of the two objects.

上图中同样了展示了一个局部变量指向堆上两个不同的对象。指向不同对象(Object1, Object5)的引用不是同一个对象。理论上，如果这两个线程有指向这两个对象的引用，那么这两个线程都能够访问Object1和Object5。但是在上图中每个线程只有指向其中一个对象的引用。

那么，在Java代码中如何反应上面的内存图呢？代码很简单，如下所示：

```java
public class MyRunnable implements Runnable() {
    public void run() {
        methodOne();
    }
    public void methodOne() {
        int localVariable1 = 45;

        MySharedObject localVariable2 =
            MySharedObject.sharedInstance;

        //... do more with local variables.

        methodTwo();
    }
    public void methodTwo() {
        Integer localVariable1 = new Integer(99);

        //... do more with local variable.
    }
}
```

```java
public class MySharedObject {

    //static variable pointing to instance of MySharedObject

    public static final MySharedObject sharedInstance =
        new MySharedObject();


    //member variables pointing to two objects on the heap

    public Integer object2 = new Integer(22);
    public Integer object4 = new Integer(44);

    public long member1 = 12345;
    public long member1 = 67890;
}
```

如果两个线程执行run()方法,那么此代码就表明了上图所示的内存分布。run方法首先调用methodOne方法，然后methodOne方法调用methodTwo方法。

在methodOne方法中定义了一个原型的局部变量localVariable1,同时定义了一个指向对象的引用localVariable2.

每个执行methodOne方法的线程都会在各自的线程栈空间中创建localVariable1和localVariable2的副本。localVariable1在两个线程中是完全独立的，仅仅存在于对应线程的栈空间中。一个线程不能看到其他线程对于localVariable1变量的修改。

执行methodOne方法的线程同样会创建localVariable2的副本。但是，这两个localVariable2的副本是指向在堆上的同一个对象。而localVariable2引用指向的对象是一个类的静态成员变量。而类的静态变量在堆中只存在一份。所以两个localVariable2指向同一个MySharedObject对象的实例，MySharedObject实例存储在堆内存上，对应图中的Object3对象。

我们在来看看methodTwo方法是如何创建localVariable1局部变量的。这个局部变量是指向一个Integer对象的引用。methodTwo方法每次都重新创建一个Integer实例。局部变量localVariable2引用是存储在对应的栈空间中，而对应的Integer对象是在堆内存中，因为每次执行methodTwo方法都会创建Integer对象，所以在堆内存中会有两个Integer对象。即对应于图中的Object1和Object5对象。

在MySharedObject中有两个long类型的局部变量，因为这些变量是类的成员变量，所以它们跟具体的对象一起保存在堆内存中。仅仅局部变量是保存在栈内存中的。

# 硬件内存架构

硬件内存跟Java内存模型是有些不一样的。为了更好地理解Java内存模型，我们需要了解硬件内存架构。下面的图简单描述了现代计算机的硬件架构：

{% asset_img java-memory-model-4.png java-memory-model %}

现代的计算机通常会有两个以上的CPU，同时这些CPU可能有多个核。这也意味着我们可以将多个线程同时运行在多个CPU上。在给定的时间点，每个CPU都能运行一个线程。即如果你的Java程序是多线程的，那么能够在多个CPU上同时运行(并行)。

在上图中我们可以看到每个CPU内部都有一系列的寄存器。CPU在寄存器上执行的操作要比在主存中的操作快。这是因为CPU能够更快的获取寄存器上的内容。

每个CPU都有一个对应的缓存内存。获取缓存中的内容要快于获取主存中的内容，但是没有获取寄存器中的内容快。CPU缓存的速度要介于寄存器和主存之间。有些CPU可能会有多级的缓存。

一个计算机包含一个主存(RAM),所有的CPU都能够获取主存中的内容。主存的容量要远远大于缓存的容量。

如果一个CPU要读取主存的内容，通常只会读取主存中部分区域的内容到CPU缓存中，然后在从缓存读取到寄存器中，之后进行计算。当CPU需要写结果到主存中，它会将寄存机中的值刷新到缓存中，然后在之后的某个时间点，在将缓存中的内容刷新到主存中。

当CPU需要存储缓存中的值时，会将缓存中的值刷新到主存中。同时CPU缓存也可以局部的刷新缓存值以及写出缓存值。

# Java内存模型和硬件内存架构之间的桥接

像前面说明的，Java内存模型和硬件内存架构是不同的。硬件内存架构不会区分线程栈和堆空间。在硬件中，线程栈和堆空间都是在主存中的。同时，部分线程栈和堆内存会出现在CPU缓存或者CPU寄存器中。下面的图进行了说明：

{% asset_img java-memory-model-5.png java-memory-model %}

当对象和变量存储在不同的内存区域时，会出现一些问题。主要的两个问题如下：

1. 共享变量的可见性(可见性，即一个线程能够及时的看到另一个线程对于共享变量的修改)
2. 竞态条件(即读取，检查，写入共享变量)

下面会依次介绍这两个问题。

## 共享变量的可见性

如果两个线程共享一个对象，但是没有采用合适的volatile关键字或者同步操作，那么当一个线程更新共享对象时，可能另一个线程并不能看到更新后的值。

想象一下，一个共享对象最初是在主存中，一个CPU上的线程读取了此对象到CPU缓存中，之后对于这个共享对象进行了修改。只要CPU缓存没有刷新到主存中，那么运行在其他CPU上的线程是不能读取到修改后的共享变量的值的。这会造成其他CPU只能看到修改之间的值。

下面的图说明了这种情况。在左边的CPU上运行的线程将共享对象读取到CPU缓存中，然后将它的count变量修改为2.由于没有将count的修改刷新到主存中，所以在右侧CPU上运行的线程不能看到这个修改。

{% asset_img java-memory-model-6.png java-memory-model %}

为了解决这个问题，我们可以使用Java中的[volatile](<http://tutorials.jenkov.com/java-concurrency/volatile.html>)关键字.volatile关键字的作用是保证每次都是从主存中读取变量的值，同时如果修改了此变量，那么此变量会立刻写回到主存中。

## 竞态条件

如果两个线程或多个线程共享一个对象，那么当多余一个线程修改此变量时，竞态条件就可能会出现。

想象一下，线程A将一个共享对象的count变量读取进CPU缓存，线程B也将count读取到另一个CPU缓存。现在线程A在count加上1，同时线程B也在count上加上1.现在变量被增加了两次，在每个CPU缓存中各一次。

如果加1操作是顺序进行的，那么count的值会加上2，然后写回到主存中。

但是，这两个操作是在没有正确同步的情况下同时进行的。尽管线程A或者B会将count修改后的值写回到主存，但是更新后的值始终比原来的值大1.

下面的图说明了上面描述的竞态条件：

{% asset_img java-memory-model-7.png java-memory-model %}

为了解决这个问题，可以使用Java的同步块，即[synchronized](<http://tutorials.jenkov.com/java-concurrency/synchronized.html>)关键字。同步块保证了在同步块中获取到的变量都是从主存中读取的，并且当线程离开同步块时，所有修改的变量会被刷新到主存中，而不管这个变量是否被volatile声明。
