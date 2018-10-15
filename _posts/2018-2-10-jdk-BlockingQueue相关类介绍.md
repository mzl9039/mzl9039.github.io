---
layout:     post
title:      "jdk BlockingQueue源码阅读"
subtitle:   ""
date:       2018-2-10 22:00:05
author:     mzl
catalog:    true
tags:
    - jdk
    - concurrent
    - BlockingQueue
---

{:toc}
# BlockingQueue

阻塞队列：能够阻塞当前试图从队列中获取元素的线程。而非阻塞队列则做不到，这是两者最大的区别。
最典型的应用场景是消费者-生产者模型。

## 阻塞队列概况

阻塞队列由 BlockingQueue 定义。jdk 1.8 中实现了该接口的主要有以下几个：
* ArrayBlockingQueue: 基于数组实现的阻塞队列，初始化时必须指定容量大小，可以指定公平锁或非公平锁，
默认非公平锁，即不保证等待时间最长的线程最先能够访问队列。
* LinkedBlockingQueue: 基于链表实现的阻塞队列，创建时若不指定容量，则默认为 Integer.MAX_VALUE
* PriorityBlockingQueue: 无界阻塞队列，会按照元素优先级对元素进行排序，按照优先级顺序出队，每次
出队的元素都是优先级最高的元素。
* DelayQueue: 基于 PriorityBlockingQueue 实现的延迟队列，也是无界队列，用于放置实现了 Delayed 接口
的对象，其中的对象只能在其到期时才能从队列中取走。因此向队列中插入时永远不会阻塞，获取时才可能阻塞。
* SynchronousQueue: 同步阻塞队列，队列大小为1。一个元素只在放到该队列中，必须有一个线程在等待获取元素。
* DelayedWorkQueue: 为 ScheduledThreadPoolExecutor 中的静态内部类，ScheduledThreadPoolExecutor 
就是通过该队列使得队列中的元素按一定顺序排列，从而使延迟任务和周期性得以顺利执行。
* BlockingDeque: 双向阻塞队列的接口
* TransferQueue: 接口，定义了另一种阻塞情况：生产者一直阻塞，直到所添加到队列的元素被某一消费者消费，
而 BlockingQueue 只需将元素添加到队列中后，生产者便会停止阻塞。

## 阻塞队列与非阻塞队列中的方法对比

### 非阻塞队列常用方法

* add(E e): 将元素 e 插入队列末尾，如果插入成功，则返回 true；如果插入失败（即队列已满），则抛出异常
* remove(): 移除队首元素，若移除成功，则返回 true; 如果移除失败（队列为空），则抛出异常
* offer(E e): 将元素 e 插入队列末尾，如果插入成功，则返回 true; 如果插入失败（即队列已满）,则返回 false
* poll(): 移除并获取队首元素，若成功，则返回队首元素； 否则返回 null
* peek(): 获取队首元素，但不移除。若成功，则返回队首元素，否则返回 null

### 阻塞队列常用方法

阻塞队列也实现了 Queue，因此也具有上述方法，并且都进行了同步处理。除此之外，还有4种很有用的方法：
* put(E e): 向队尾插入元素，如果队列满，则等待；
* take(): 从队首获取元素，如果队列为空，则等待；
* offer(E e, long timeout, TimeUnit unit): 向队尾插入元素，如果队列满，则等待一定时间，当时间期限
达到时，如果还没有插入成功，则返回 false，否则返回 true
* poll(long timeout, TimeUnit unit): 从队首获取元素，如果队列为空，则等待一段时间，当时间期限达到时，
如果取不到，则返回 null; 否则返回取到的元素.

## 各阻塞队列实现原理

### BlockingQueue 接口

[BlockingQueue]()

### ArrayBlockingQueue

[ArrayBlockingQueue]()

### LinkedBlockingQueue

[LinkedBlockingQueue]()

### PriorityBlockingQueue

[PriorityBlockingQueue]()

### DelayQueue

[DelayQueue]()

### SynchronousQueue

[SynchronousQueue]()

### DelayedWorkQueue

[DelayedWorkQueue]()

### TransferQueue

[TransferQueue]()

## 转载和引用
* [BlockingQueue源码解析jdk1.8](http://blog.csdn.net/qq_38989725/article/details/73298856)
