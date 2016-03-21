---
layout: post
title: 知识点总结
category: 学习
tags: KIDLOSERME
keywords: NULL
description: 
---
##HaspMap原理
根据key的hashCode与Enty[]长度取模获取index来决定放入Enty中的位置，index重复则使用Entry.next在同一个index中放入多个值
具体可看：http://www.cnblogs.com/xwdreamer/archive/2012/05/14/2499339.html 介绍

---

##HashMap和HashTable的区别
HaspMap继承AbstractMap ，HashTable继承Dictionary
HashMap的方法不是同步的，HashTable的方法是同步的
HashMap允许key和value为Null，HashTable不允许key和value为null
详见：http://blog.csdn.net/shohokuf/article/details/3932967

---

##HashMap TreeMap  LinkedHashMap 元素顺序
HashMap不保证元素的插入顺序，TreeMap默认会按照key的升序排序TreeMap支持自定义排序，LinkedHashMap按照插入顺序排序

---

##List子类ArrayList LinkedList  Vector
LinkedList在 add和remove 上更快,而在get上更慢.
List接口下一共实现了三个类：ArrayList，Vector，LinkedList。LinkedList就不多说了，它一般主要用在保持数据的插入顺序的时候。
ArrayList和Vector都是用数组实现的，主要有这么三个区别：
 - 1、Vector是多线程安全的，而ArrayList不是，这个可以从源码中看出，Vector类中的方法很多有synchronized进行修饰，这样就导致了Vector在效率上无法与ArrayList相比；
 - 2、两个都是采用的线性连续空间存储元素，但是当空间不足的时候，两个类的增加方式是不同的，很多网友说Vector增加原来空间的一倍，ArrayList增加原来空间的50%，其实也差不多是这个意思，不过还有一点点问题可以从源码中看出，一会儿从源码中分析。
 - 3、Vector可以设置增长因子，而ArrayList不可以，最开始看这个的时候，我没理解什么是增量因子，不过通过对比一下两个源码理解了这个，先看看两个类的构造方法：

---

##HashSet
HashSet内部实际上是一个HashMap，看代码

```java
transient HashMap<E, HashSet<E>> backingMap;
看添加代码：
@Override
public boolean add(E object) {
    return backingMap.put(object, this) == null;
}
//所以Set几何不允许重复的元素存在
```

---

##Map List Set
 - Map内部是一个Entry数组；
 - List内部是一个Object数组；
 - Set内部是一个Map集合，map的key集合就是Set的Value集合

---

##关键字transient：
 - 1）一旦变量被transient修饰，变量将不再是对象持久化的一部分，该变量内容在序列化后无法获得访问。
 - 2）transient关键字只能修饰变量，而不能修饰方法和类。注意，本地变量是不能被transient关键字修饰的。变量如果是用户自定义类变量，则该类需要实现Serializable接口。
 - 3）被transient关键字修饰的变量不再能被序列化，一个静态变量不管是否被transient修饰，均不能被序列化。

---

##一个关于线程的问题
假如有Thread1、Thread2、Thread3、Thread4四条线程分别统计C、D、E、F四个盘的大小，所有线程都统计完毕交给Thread5线程去做汇总，应当如何实现？
可以使用并发包下的CountDownLatch实现
http://www.cnblogs.com/dolphin0520/p/3920397.html

---

##wait和notify：
执行wait、notify必须在同步块内部，这两个动作都是针对某一个对象的，比如对象A在线程1中执行了wait，那么线程1就会停留在此处，类似于阻塞，然后A在线程2中调用了notify，并且线程2执行结束之后线程1继续执行，如果对象A不在某处调用notify，线程1会一直停留在wait那一行处不继续执行。或者A调用wait的时候传一个等待时间，如果在这个等待时间内notify没有被调用，线程1会恢复执行。如果对象A在多个线程调用wait，那么必须执行notifyAll才能唤醒所有等待的线程，否则只会喊醒其中一个。

---

##线程通信
线程之间的通信机制:(共享内存、消息传递)
深入理解Java内存模型http://www.infoq.com/cn/articles/java-memory-model-1

---

##Volatile的官方定义
Java语言规范第三版中对volatile的定义如下： java编程语言允许线程访问共享变量，为了确保共享变量能被准确和一致的更新，线程应该确保通过排他锁单独获得这个变量。Java语言提供了volatile，在某些情况下比锁更加方便。如果一个字段被声明成volatile，java线程内存模型确保所有线程看到这个变量的值是一致的

---

##线程池
为什么要使用线程池
避免频繁地创建和销毁线程，达到线程对象的重用。另外，使用线程池还可以根据项目灵活地控制并发的数目。
从Java5开始，Java提供了自己的线程池。每次只执行指定数量的线程，java.util.concurrent.ThreadPoolExecutor
public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,BlockingQueue<Runnable> workQueue,RejectedExecutionHandler handler);   

###参数介绍：
 - corePoolSize 核心线程数，指保留的线程池大小（不超过maximumPoolSize值时，线程池中最多有corePoolSize 个线程工作）。 
 - maximumPoolSize 指的是线程池的最大大小（线程池中最大有corePoolSize 个线程可运行）。 
 - keepAliveTime 指的是空闲线程结束的超时时间（当一个线程不工作时，过keepAliveTime 长时间将停止该线程）。 
 - unit 是一个枚举，表示 keepAliveTime 的单位（有NANOSECONDS, MICROSECONDS, MILLISECONDS, SECONDS, MINUTES, HOURS, DAYS，7个可选值）。 
 - workQueue 表示存放任务的队列（存放需要被线程池执行的线程队列）。 
 - handler 拒绝策略（添加任务失败后如何处理该任务）

###要点：
 - 1、线程池刚创建时，里面没有一个线程。任务队列是作为参数传进来的。不过，就算队列里面有任务，线程池也不会马上执行它们。
 - 2、当调用 execute() 方法添加一个任务时，线程池会做如下判断：
   - a. 如果正在运行的线程数量小于 corePoolSize，那么马上创建线程运行这个任务；
   - b. 如果正在运行的线程数量大于或等于 corePoolSize，那么将这个任务放入队列。
   - c. 如果这时候队列满了，而且正在运行的线程数量小于 maximumPoolSize，那么还是要创建线程运行这个任务；
   - d. 如果队列满了，而且正在运行的线程数量大于或等于 maximumPoolSize，那么线程池会抛出异常，告诉调用者“我不能再接受任务了”。
 - 3、当一个线程完成任务时，它会从队列中取下一个任务来执行。
 - 4、当一个线程无事可做，超过一定的时间（keepAliveTime）时，线程池会判断，如果当前运行的线程数大于corePoolSize，那么这个线程就被停掉。所以线程池的所有任务完成后，它最终会收缩到 corePoolSize 的大小。
     
这个过程说明，并不是先加入任务就一定会先执行。假设队列大小为4，corePoolSize为2，maximumPoolSize为6，那么当加入15个任务时，执行的顺序类似这样：首先执行任务 1、2，然后任务3~6被放入队列。这时候队列满了，任务7、8、9、10 会被马上执行，而任务 11~15则会抛出异常。最终顺序是：1、2、7、8、9、10、3、4、5、6。当然这个过程是针对指定大小的ArrayBlockingQueue<Runnable>来说，如果是LinkedBlockingQueue<Runnable>，因为该队列无大小限制，所以不存在上述问题。


Executors.newCachedThreadPool();        //创建一个缓冲池，缓冲池容量大小为Integer.MAX_VALUE<br>
Executors.newSingleThreadExecutor();   //创建容量为1的缓冲池<br>
Executors.newFixedThreadPool(int);    //创建固定容量大小的缓冲池<br>
