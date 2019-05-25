---
title: Thread Signaling
date: 2019-05-14 10:52:48
categories:
- java
tags:
- 并发
copyright:
top:
type:
---

# 线程信号量及wait，notify方法
本篇主要介绍线程之间如何进行信号的通知。同时介绍wait，notify底层的一些实现。

## 通过共享对象进行信号通知
最简单的进行线程之间通知的方式就是采用共享变量的方式。比如下面的代码
```java
public class MySignal{
  protected boolean hasDataToProcess = false;
  public synchronized boolean hasDataToProcess(){
    return this.hasDataToProcess;
  }
  public synchronized void setHasDataToProcess(boolean hasData){
    this.hasDataToProcess = hasData;  
  }
}
```
线程A和线程B共享同一个MySingal实例，当线程A处理好数据后，可以设置hasDataToProcess属性为true，然后线程B获取到此属性。从而完成线程之间的信号通知。当然，如果线程A和线程B不是在同一个MySingal实例上进行的，则不能进行信号的传递。

## 忙等待
在采用MySingal的例子中，一般会采用下面的代码来判断一个线程是否可以进行处理了。
```java
protected MySignal sharedSignal = new MySingal()
while(!sharedSignal.hasDataToProcess()){
  //do nothing... busy waiting
}
```
从代码中可以看到，检测hasDataToProcess属性是在一个while循环中，如果hasDataToProcess为false，这就会造成线程一直在执行while语句，造成忙等待。这会造成CPU资源的浪费。

## wait，notify，notifyAll使用
一般在Java中，我们一般会采用wait，notify或notifyAll进行线程之间信号的传递。线程调用某一个对象上的wait方法前，必须首先获取该对象上的锁，才能执行此对象上的wait方法。在调用notify或者notifyAll之前，也是要获取对应对象上的锁。调用wait方法时，会释放此对象上的锁，线程进入阻塞状态，等待信号。而调用notify后，会随机唤醒一个同对象上的线程，但是必须是退出了notify对应的synchronized块后，被唤醒的线程才能继续执行，因为被唤醒的线程还要获取对象上的锁。如下面的代码所示：
```java
public class MonitorObject{
}
public class MyWaitNotify{
  MonitorObject myMonitorObject = new MonitorObject();

  public void doWait(){
    synchronized(myMonitorObject){
      try{
        myMonitorObject.wait();
      } catch(InterruptedException e){...}
    }
  }

  public void doNotify(){
    synchronized(myMonitorObject){
      myMonitorObject.notify();
    }
  }
}
```
从代码中，可以看到在myMonitorObject上调用wait方法之前，会先获取myMonitorObject上的锁。在调用notify方法之前，也是要先获取myMonitorObject上的锁。在下面的内容中我会简单介绍wait和notify的底层原理。

### wait notify notifyAll在Object源码中的介绍
在JDK1.8中，Object对象中有三个wait(),wait(long timeout),wait(long timeout, int nanos)这三个方法，它们的主要区别是后两个方法增加了等待的时间。
下面是JDK1.8中关于wait(long timeout)方法的描述：
> /**
     * Causes the current thread to wait until either another thread invokes the
     * {@link java.lang.Object#notify()} method or the
     * {@link java.lang.Object#notifyAll()} method for this object, or a
     * specified amount of time has elapsed.
     * <p>
     * The current thread must own this object's monitor.
     * <p>
     * This method causes the current thread (call it <var>T</var>) to
     * place itself in the wait set for this object and then to relinquish
     * any and all synchronization claims on this object. Thread <var>T</var>
     * becomes disabled for thread scheduling purposes and lies dormant
     * until one of four things happens:
     * <ul>
     * <li>Some other thread invokes the {@code notify} method for this
     * object and thread <var>T</var> happens to be arbitrarily chosen as
     * the thread to be awakened.
     * <li>Some other thread invokes the {@code notifyAll} method for this
     * object.
     * <li>Some other thread {@linkplain Thread#interrupt() interrupts}
     * thread <var>T</var>.
     * <li>The specified amount of real time has elapsed, more or less.  If
     * {@code timeout} is zero, however, then real time is not taken into
     * consideration and the thread simply waits until notified.
     * </ul>
     * The thread <var>T</var> is then removed from the wait set for this
     * object and re-enabled for thread scheduling. It then competes in the
     * usual manner with other threads for the right to synchronize on the
     * object; once it has gained control of the object, all its
     * synchronization claims on the object are restored to the status quo
     * ante - that is, to the situation as of the time that the {@code wait}
     * method was invoked. Thread <var>T</var> then returns from the
     * invocation of the {@code wait} method. Thus, on return from the
     * {@code wait} method, the synchronization state of the object and of
     * thread {@code T} is exactly as it was when the {@code wait} method
     * was invoked.
     * <p>
     * A thread can also wake up without being notified, interrupted, or
     * timing out, a so-called <i>spurious wakeup</i>.  While this will rarely
     * occur in practice, applications must guard against it by testing for
     * the condition that should have caused the thread to be awakened, and
     * continuing to wait if the condition is not satisfied.  In other words,
     * waits should always occur in loops, like this one:
     * <pre>
     *     synchronized (obj) {
     *         while (&lt;condition does not hold&gt;)
     *             obj.wait(timeout);
     *         ... // Perform action appropriate to condition
     *     }
     * </pre>
     * (For more information on this topic, see Section 3.2.3 in Doug Lea's
     * "Concurrent Programming in Java (Second Edition)" (Addison-Wesley,
     * 2000), or Item 50 in Joshua Bloch's "Effective Java Programming
     * Language Guide" (Addison-Wesley, 2001).
     *
     * <p>If the current thread is {@linkplain java.lang.Thread#interrupt()
     * interrupted} by any thread before or while it is waiting, then an
     * {@code InterruptedException} is thrown.  This exception is not
     * thrown until the lock status of this object has been restored as
     * described above.
     *
     * <p>
     * Note that the {@code wait} method, as it places the current thread
     * into the wait set for this object, unlocks only this object; any
     * other objects on which the current thread may be synchronized remain
     * locked while the thread waits.
     * <p>
     * This method should only be called by a thread that is the owner
     * of this object's monitor. See the {@code notify} method for a
     * description of the ways in which a thread can become the owner of
     * a monitor.
     *
     * @param      timeout   the maximum time to wait in milliseconds.
     * @throws  IllegalArgumentException      if the value of timeout is
     *               negative.
     * @throws  IllegalMonitorStateException  if the current thread is not
     *               the owner of the object's monitor.
     * @throws  InterruptedException if any thread interrupted the
     *             current thread before or while the current thread
     *             was waiting for a notification.  The <i>interrupted
     *             status</i> of the current thread is cleared when
     *             this exception is thrown.
     * @see        java.lang.Object#notify()
     * @see        java.lang.Object#notifyAll()
     */
    public final native void wait(long timeout) throws InterruptedException;

从代码的注释中，我们可以获得以下一些内容：
1. 调用wait的线程一定要获取对象上的monitor，在调用wait后，会释放该对象上的锁。
2. 发生以下四种情况线程会被唤醒：
    - 其他的线程在相同的对象上调用了notify
    - 其他的线程在相同的对象上调用了notifyAll
    - 其他的线程终止了当前线程(interrupt方法)，会扔出InterruptedException异常
    - 指定的最大等待时间到了
3. 当线程被唤醒时，当前的线程会从对象的等待集合中移除，重新进入线程调度阶段。之后会跟其他线程竞争获取对象上的锁。
4. 线程可能在没有上述四种情况发生的时候，被唤醒。这被称为伪唤醒，下面会有详细介绍。

### 浅析Object monitor的底层原理

在JVM实现获取对象上的锁，是通过monitor进行实现的。图示如下：
{% asset_img monitor.png java-memory-model %}
当一个线程需要获取 Object 的锁时，会被放入 EntrySet 中进行等待，如果该线程获取到了锁，成为当前锁的 owner。如果根据程序逻辑，一个已经获得了锁的线程缺少某些外部条件，而无法继续进行下去（例如生产者发现队列已满或者消费者发现队列为空），那么该线程可以通过调用 wait 方法将锁释放，进入 wait set 中阻塞进行等待，其它线程在这个时候有机会获得锁，去干其它的事情，从而使得之前不成立的外部条件成立，这样先前被阻塞的线程就可以重新进入 EntrySet 去竞争锁。

### notify和notifyAll区别

乍一看，notify和notifyAll的区别很简单，就是notify只能随机选择一个处于等待状态的线程进行唤醒；而notifyAll可以唤醒所有处于等待状态下的线程，但是也是只有一个线程能够继续执行。
如果结合上面的图片，我们就能更好地进行理解。当线程在对象上调用notify方法时，随机选择一个处于等待状态的线程，并且把该线程放置在该对象的EntrySet列表中。而如果调用的是notifyAll方法时，所有的处于等待状态下的线程都会进入到EntrySet中，从而多个线程进行对象上的锁。notify有点类似于网络中的单播，而notifyAll类似于多播。

## 丢失信号

设想一下这种情况，有一个线程首先调用了notify方法，然后其他的线程调用了wait方法。那么处于wait下的线程将不会接受到之前线程发送的notify信号。如下面代码所示，[代码地址](https://github.com/lightnine/daydayup/blob/master/src/main/java/com/leon/concurrent/waitNotify/WaitNotifyMissSingal.java)：
```java
public class WaitNotifyMissSingal {
    public static void main(String[] args) {
        final Object lock = new Object();
        new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("thread B is waiting to get lock");
                synchronized (lock) {
                    System.out.println("thread B get lock");
                    try {
                        TimeUnit.SECONDS.sleep(5);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    lock.notify();
                    System.out.println("thread B do notify method");
                }
            }
        }).start();

        new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("thread A is waiting to get lock");
                synchronized (lock) {
                    try {
                        System.out.println("thread A get lock");
                        TimeUnit.SECONDS.sleep(1);
                        System.out.println("thread A do wait method");
                        lock.wait();
                        System.out.println("wait end");
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }).start();
    }
}
```
运行上面的代码，程序会一直运行下去。那么如何解决这个问题呢，其实在Object的wait的方法注释中也有对应的说明。我们可以把通知信号保存在信号类的成员变量中。[代码地址](https://github.com/lightnine/daydayup/blob/master/src/main/java/com/leon/concurrent/waitNotify/MyWaitNotify2.java)
```java
public class MyWaitNotify2{
  MonitorObject myMonitorObject = new MonitorObject();
  boolean wasSignalled = false;

  public void doWait(){
    synchronized(myMonitorObject){
      if(!wasSignalled){
        try{
          myMonitorObject.wait();
         } catch(InterruptedException e){...}
      }
      //clear signal and continue running.
      wasSignalled = false;
    }
  }

  public void doNotify(){
    synchronized(myMonitorObject){
      wasSignalled = true;
      myMonitorObject.notify();
    }
  }
}
```
在上面的代码中，将信号保存在了wasSingalled变量中，只要调用了notify方法，就是将wasSignalled设置为true，表示有线程执行了notify。在doWait方法中，如果wasSignalled为false，则当前的线程会执行wait方法，进入等待状态；而当wasSignalled为true时，不会执行wait方法，只会将wasSignalled设置为false。

## 伪唤醒

因为一些底层操作系统的原因，具体的可以查看unix操作系统相关内容。简单理解，就是当线程没有接受到唤醒信号时，而线程被错误的唤醒。为了防止伪唤醒，一定要在while循环中检查信号变量的值。这样的循环也叫自旋锁。代码如下：
```java
public class MyWaitNotify3{

  MonitorObject myMonitorObject = new MonitorObject();
  boolean wasSignalled = false;

  public void doWait(){
    synchronized(myMonitorObject){
      while(!wasSignalled){
        try{
          myMonitorObject.wait();
         } catch(InterruptedException e){...}
      }
      //clear signal and continue running.
      wasSignalled = false;
    }
  }

  public void doNotify(){
    synchronized(myMonitorObject){
      wasSignalled = true;
      myMonitorObject.notify();
    }
  }
}
```
## 不要在常量字符串或全局对象上调用wait方法
请看下面的例子
```java
public class MyWaitNotify{

  String myMonitorObject = "";
  boolean wasSignalled = false;

  public void doWait(){
    synchronized(myMonitorObject){
      while(!wasSignalled){
        try{
          myMonitorObject.wait();
         } catch(InterruptedException e){...}
      }
      //clear signal and continue running.
      wasSignalled = false;
    }
  }

  public void doNotify(){
    synchronized(myMonitorObject){
      wasSignalled = true;
      myMonitorObject.notify();
    }
  }
}
```
从代码中我们可以看到，在上锁时是对myMonitorObject对象上锁，也是在myMonitorObject上调用wait方法，而myMonitorObject是一个空的字符串。我们知道，JVM会将相同的字符串看成同一个对象。也就说说，如果有两个MyWaitNotify实例，则这两个实例中的myMonitorObject是指向同一个对象的。那么一个在实例MyWaitNotify上调用wait的线程会被在另一个实例MyWaitNotify上调用doNotify方法的线程唤醒。下面的图表示了实例的成员指向了相同的对象。
{% asset_img strings-wait-notify.png strings-wait-notify %}
在上图中，4个线程是在相同的String常量上调用wait和notify方法，但是信号wasSignalled仍然是单独存放在对应的实例对象上。即一个在MyWaitNotify1实例上调用doNotify方法的线程可能会唤醒在MyWaitNotify2实例上等待的线程，但是唤醒信号仍然是单独保存在MyWaitNotify1实例中。
如果仔细看上面的程序，发现当在第二个MyWaitNotify2实例上调用doNotify方法时，会唤醒线程A或者B，但是由于在while循环中的wasSignalled变量，对于MyWaitNotify1实例仍然是false。所以被唤醒的线程A或者B从wait启动，但是会再次进入while循环调用wait，再次进入阻塞状态。这跟伪唤醒很像。
由于在doNotify方法中调用的notify方法，此方法不像notifyAll方法，notify方法只会唤醒一个线程，如果是线程C调用的doNotify，本来想唤醒的是线程D。但是有可能会错误的唤醒线程A或B，并且线程A或B会修改对应实例上的wasSignalled变量。而发给D的信号就丢失了，就有点像信号丢失的情况。如果将doNotify方法中的notify替换成notifyAll就不会有这个问题。但是这是一个坏主意，当应用仅仅只需要唤醒一个线程时，没有任何理由要把所有的线程都唤醒。
注意，对于wait/notify的情况，永远不要使用全局的对象例如string常量。在wait和notify上使用的对象针对对应的实例必须是独一的。

## 参考
1. http://tutorials.jenkov.com/java-concurrency/thread-signaling.html
2. http://www.php.cn/java-article-410323.html