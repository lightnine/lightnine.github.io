---
title: java synchronized keyword
date: 2019-05-10 19:39:54
categories:
- java
tags:
- 并发
copyright:
top:
type:
---
# java synchronized关键字
在平常进行java并发程序的开发过程中，synchronized关键字的出现频率很高。但是synchronized底层是如何实现的，synchronized都有哪些具体的用法。本篇将会在下面进行详细讲解。
synchronized关键字主要是用来进行同步操作。synchronized关键字修饰的内容，每次都只能有一个线程进入，如果其他线程想要进入相同的代码块，那么必须等前一个线程释放代码块对应的锁，其他的线程才能进入此代码块。但是同一个线程能够多次进入一个相同的同步块，也就是synchronized具有可重入锁的特性。
总的来说，在java中，主要在三个地方使用synchronized关键字
1. 类的实例方法
2. 类的静态方法
3. 修饰部分代码块

## synchronized用在实例方法上
```java
public class MyClassInstance {
    private int count;
    public synchronized void add(int value){
      this.count += value;
  }
}
```
在上面的代码中展示了，synchronized应用在实例方法的方式。当synchronized用在实例方法上时，每当线程进入此方法时，会尝试获取对应的类实例上的锁(注意是类实例)。如果没有其他的线程持有此实例上的锁，那么线程会获取此实例锁，然后运行此方法。
下面的图显示了使用javap进行反编译后的内容(运行：`javap -v MyClassInstance.class`)：
{% asset_img MyClassInstance.png java-memory-model %}
图中仅仅展示了add方法反编译后的内容，可以看到在方法的flags标记中出现了ACC_SYNCHRONIZED。这个就是表明前面方法是用synchronized关键字修饰的。这个跟同步代码块的反编译结果是不同的。但是在线程执行同步操作时，都是要获取对应对象上的锁。
## synchronized用在类的静态方法上
```java
public class MyClassStatic {
    public static synchronized void add(int value) {
        System.out.print(value);
    }
}
```
上面代码展示了如何在静态方法上使用synchronized关键字。当线程尝试进入类的静态方法时，会尝试获取类上的锁(注意是类上的锁，跟类实例上的锁不同的东西)。如果没有其他线程持有此类上的锁，那么当前线程会获取此类上的锁，然后运行此方法。
下面展示了反编译的结果(运行命令:`javap -v MyClassStatic.class`)
{% asset_img MyClassStatic.png java-memory-model %}
可以从反编译的结果中看到在方法的flags中也是包含了ACC_SYNCHRONIZED.
## synchronized用在代码块上
如果synchronized关键字用在代码块上，会在其之后括号中的对象获取锁。
```java
public class MyClass {
    public void log(String msg) {
        synchronized(this) {
            System.out.println(msg);
        }
    }
    public static void func(String msg) {
        synchronized(MyClass.class) {
            System.out.print(msg);
        }
    }
}
```
在上面的代码中，在方法log中，当线程进入log方法，然后执行synchronized关键字修饰的代码块。线程会尝试获取当前类实例上的锁，因为括号中使用的是this关键字。而在func方法中，线程会尝试获取MyClass类的锁。注意这两个锁是不同的。所以一个线程可以执行log中的同步代码块，而同时另一个线程也可以执行func中的同步代码块。
反编译结果如下(运行命令:`javap -v MyClass.class`):
{% asset_img MyClass1.png java-memory-model %}
可以看到在方法的签名中是没有ACC_SYNCHRONIZED的。但是在代码中出现了monitorenter和monitorexit。
**monitorenter**：
> Each object is associated with a monitor. A monitor is locked if and only if it has an owner. The thread that executes monitorenter attempts to gain ownership of the monitor associated with objectref, as follows:
> • If the entry count of the monitor associated with objectref is zero, the thread enters the monitor and sets its entry count to one. The thread is then the owner of the monitor.
> • If the thread already owns the monitor associated with objectref, it reenters the monitor, incrementing its entry count.
> • If another thread already owns the monitor associated with objectref, the thread blocks until the monitor's entry count is zero, then tries again to gain ownership.
引用中的内容来自JVM规范。这段话的大概意思为：

每个对象有一个监视器锁（monitor）。当monitor被占用时就会处于锁定状态，线程执行monitorenter指令时尝试获取monitor的所有权，过程如下：
1. 如果monitor的进入数为0，则该线程进入monitor，然后将进入数设置为1，该线程即为monitor的所有者。
2. 如果线程已经占有该monitor，只是重新进入，则进入monitor的进入数加1.
3. 如果其他线程已经占用了monitor，则该线程进入阻塞状态，直到monitor的进入数为0，再重新尝试获取monitor的所有权。

**monitorexit**:
> The thread that executes monitorexit must be the owner of the monitor associated with the instance referenced by objectref.
> The thread decrements the entry count of the monitor associated with objectref. If as a result the value of the entry count is zero, the thread exits the monitor and is no longer its owner. Other threads that are blocking to enter the monitor are allowed to attempt to do so.

这段话的大概意思为：
执行monitorexit的线程必须是objectref所对应的monitor的所有者。
指令执行时，monitor的进入数减1，如果减1后进入数为0，那线程退出monitor，不再是这个monitor的所有者。其他被这个monitor阻塞的线程可以尝试去获取这个 monitor 的所有权。 
通过这两段描述，我们应该能很清楚的看出Synchronized的实现原理，Synchronized的语义底层是通过一个monitor的对象来完成，其实wait/notify等方法也依赖于monitor对象，这就是为什么只有在同步的块或者方法中才能调用wait/notify等方法，否则会抛出java.lang.IllegalMonitorStateException的异常的原因。
一个monitorenter都会对应有一个monitorexit。但是我们从反编译的结果中，可以看到多出了一个monitorexit，即第18行。因为在synchronized中的代码遇到异常时，会释放锁。第一个 monitorexit 指令如果正确执行，会走到下面的 goto 指令，直接跳转到 21 行 return，而如果发生异常，下面的 astore_3 和 aload_2 指令会继续执行异常问题，下一步会继续执行 monitorexit 指令退出同步。

当然，有时候，我们也可以这样用synchronized关键字修饰代码块
```java
public class MyClass2 {
    private Object obj = new Object();
    private static Object obj2 = new Object();
    public void log(String msg) {
        synchronized(obj) {
            System.out.print(msg);
        }
    }
    public static void func(String msg) {
        synchronized(obj2) {
            System.out.print(msg);
        }
    }
}
```
在上面的代码中，log方法中的同步块是对obj实例进行加锁，注意每个MyClass2实例中的obj都是不同的。而func中的同步块是对obj2静态成员进行加锁。
