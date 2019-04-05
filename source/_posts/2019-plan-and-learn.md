---
title: 2019-plan-and-learn
date: 2019-01-05 17:08:14
categories:
 - plan
tags:
 - 2019
copyright:
top:
type:
---

# 2019计划

激励[左耳朵耗子](http://www.ituring.com.cn/article/9174)

## 目标

1. 掌握tensorflow的使用
2. 掌握Python以及常用库(numpy,matplotlib,pandas)的使用
3. 掌握常用机器学习算法(SVM,AdaBoost,LightGBM)的使用以及理论
4. 掌握深度学习常用算法应用和理论
5. 掌握英语单词5000个,提高自己的英语发音
6. spark大数据分析技术
7. linux相关技术
8. Java相关技术

## 具体实现

今天看了一篇虎扑上面<非CS专业转行机器学习/人工智能阶段性成功，分享一点个人经验吧>的帖子，感触比较多，摘一点帖子里面的内容，为之后的学习提供一定的建议。
楼主的情况是没有人工智能基础的，为了找到人工智能的工作准备了两年时间。

编程：掌握一门语言，要达到非常熟悉的阶段。
数据结构：掌握常用的数据结构，多做leetcode上面的编程题目，至少要刷一下easy和medium模式的题目，写的时候考虑test case
机器学习：
    1. andrew ng的machine learning，仔细看课件和习题
    2. hands on machine learning with scikit-learn and tensorflow，并且在pythhon中实践还有课后习题
    3. bishop的pattern recognition and machine learning
    4. 李彦宏的机器学习和深度学习(自己加的)
    5. 李航的统计机器学习（自己加的）

## Java相关技术

参考了[v2ex](https://www.v2ex.com/t/546203) 或者[github](https://github.com/farmerjohngit/myblog)
这里做了一个思维导图
{% asset_img Java相关知识学习思维导图.PNG Java相关知识学习思维导图 %}
下面进行思维导图的一些解释。**同时对这些技能点进行查漏补缺，同时也会在之后的过程中添加更多的技能点**

### 集合

集合主要是java.util包下的非线程安全和线程安全集合

#### 非线程安全

1. List： ArrayList与LinkedList的实现和区别
2. Map：
    - HashMap:了解其数据结构，源码，hash冲突如何解决(链表和红黑树)，扩容时机，扩容时避免rehash优化
    - LinkedHashMap:了解基本原理，哪两种有序，如何实现LRU
    - TreeMap：了解数据结构，了解其key对象为什么必须要实现Compare接口，如何用它实现一致性哈希
3. Set：基本上是由map实现，简单看看就好

**常见问题**

- hashmap 如何解决 hash 冲突，为什么 hashmap 中的链表需要转成红黑树？
- hashmap 什么时候会触发扩容？
- jdk1.8 之前并发操作 hashmap 时为什么会有死循环的问题？
- hashmap 扩容时每个 entry 需要再计算一次 hash 吗？
- hashmap 的数组长度为什么要保证是 2 的幂？
- 如何用 LinkedHashMap 实现 LRU ？
- 如何用 TreeMap 实现一致性 hash ？

#### 线程安全的集合

1. Collection.synchronized:了解其实现原理
2. CopyOnWriteArrayList：了解写时复制机制，了解其适用场景，思考为什么没有ConcurrentArrayList
3. ConcurrentHashMap：了解实现原理，扩容时做的优化，与HashTable的对比
4. BlockingQueue：了解 LinkedBlockingQueue、ArrayBlockingQueue、DelayQueue、SynchronousQueue

**常见问题**

- ConcurrentHashMap 是如何在保证并发安全的同时提高性能？
- ConcurrentHashMap 是如何让多线程同时参与扩容？
- LinkedBlockingQueue、DelayQueue 是如何实现的？
- CopyOnWriteArrayList 是如何保证线程安全的？

### 并发

1. synchronized：了解偏向锁、轻量级锁、重量级锁的概念以及升级机制、以及和 ReentrantLock 的区别
2. CAS：了解 AtomicInteger 实现原理、CAS 适用场景、如何实现乐观锁
3. AQS：了解 AQS 内部实现、及依靠 AQS 的同步类比如 ReentrantLock、Semaphore、CountDownLatch、CyclicBarrier 等的实现
4. ThreadLocal：了解 ThreadLocal 使用场景和内部实现
5. ThreadPoolExecutor：了解线程池的工作原理以及几个重要参数的设置

**常见问题**

- synchronized 与 ReentrantLock 的区别？
- 乐观锁和悲观锁的区别？
- 如何实现一个乐观锁？
- AQS 是如何唤醒下一个线程的？
- ReentrantLock 如何实现公平和非公平锁是如何实现？
- CountDownLatch 和 CyclicBarrier 的区别？各自适用于什么场景？
- 适用 ThreadLocal 时要注意什么？比如说内存泄漏?
- 说一说往线程池里提交一个任务会发生什么？
- 线程池的几个参数如何设置？
- 线程池的非核心线程什么时候会被释放？
- 如何排查死锁？

### 引用

了解 Java 中的软引用、弱引用、虚引用的适用场景以及释放机制

**常见问题**

- 软引用什么时候会被释放
- 弱引用什么时候会被释放

### 类加载

了解双亲委派机制

**常见问题**

- 双亲委派机制的作用？
- Tomcat 的 classloader 结构
- 如何自己实现一个 classloader 打破双亲委派

### IO

了解 BIO 和 NIO 的区别、了解多路复用机制

**常见问题**

- 同步阻塞、同步非阻塞、异步的区别？
- select、poll、eopll 的区别？
- java NIO 与 BIO 的区别？
- refactor 线程模型是什么?

### JVM

了解GC和内存区域

1. 垃圾回收基本原理、几种常见的垃圾回收器的特性、重点了解 CMS （或 G1 ）以及一些重要的参数
2. 能说清 jvm 的内存划分

**常见问题**

- CMS GC 回收分为哪几个阶段？分别做了什么事情？
- CMS 有哪些重要参数？
- Concurrent Model Failure 和 ParNew promotion failed 什么情况下会发生？
- CMS 的优缺点？
- 有做过哪些 GC 调优？
- 为什么要划分成年轻代和老年代？
- 年轻代为什么被划分成 eden、survivor 区域？
- 年轻代为什么采用的是复制算法？
- 老年代为什么采用的是标记清除、标记整理算法
- 什么情况下使用堆外内存？要注意些什么？
- 堆外内存如何被回收？
- jvm 内存区域划分是怎样的？

### Spring

bean 的生命周期、循环依赖问题、spring cloud （如项目中有用过）、AOP 的实现、spring 事务传播

**常见问题**

- java 动态代理和 cglib 动态代理的区别（经常结合 spring 一起问所以就放这里了）
- spring 中 bean 的生命周期是怎样的？
- 属性注入和构造器注入哪种会有循环依赖的问题？

### Mysql

事务隔离级别、锁、索引的数据结构、聚簇索引和非聚簇索引、最左匹配原则、查询优化（ explain 等命令）

**常见问题**

- Mysql(innondb 下同) 有哪几种事务隔离级别？
- 不同事务隔离级别分别会加哪些锁？
- mysql 的行锁、表锁、间隙锁、意向锁分别是做什么的？
- 说说什么是最左匹配？
- 如何优化慢查询？
- mysql 索引为什么用的是 b+ tree 而不是 b tree、红黑树
- 分库分表如何选择分表键
- 分库分表的情况下，查询时一般是如何做排序的？

### 算法

准备一下leetcode上的算法（easy，medium）