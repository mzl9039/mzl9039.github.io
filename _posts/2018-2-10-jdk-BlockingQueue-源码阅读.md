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

BlockingQueue 是一个 Queue，添加了一些支持的操作：
1. 获取元素时，若队列为空，则等待
2. 存储元素时，若队列已满，则等待

BlockingQueue 支持 4 种形式的方法，通过不同的方式，来处理当前无法立刻满足但将来可以满足的队列操作：
1. 抛出异常
2. 返回特定值(如 null 或 false)
3. 阻塞当前线程直到操作成功
4. 只阻塞给定的最大时间，如果超过时间仍不满足则放弃。

|  |Throws exception|Special value|Blocks|Times out|
|:---:|:---:|:---:|:---:|:---:|
|Insert|add(e)|offer(e)|put(e)|offer(e, time, unit)|
|Remove|remove()|poll()|take()|poll(time, unit)|
|Examine|element()|peek()|not applicable|not applicable|

**注意事项**
* BlockingQueue 不接受空元素 null，否则会抛出 NullPointerException 异常
* BlockingQueue 有容量限制，若队列已满，则任何元素新元素在不阻塞的情况下，都不能 put 到队列里
* BlockingQueue 被设计用于生产者-消费者队列，但另外支持了 Collections 接口，因为支持 remove 元素，但这样的操作是很低效的，而且只偶尔使用，例如 a queued message is cancelled.
* BlockingQueue 是线程安全的，所有的队列方法都会自动应用内部锁或有同步控制。然而，批量的 Collections 操作（如 addAll/containsAll/retainAll）没有自动加锁的。因此可能出现如下情况：使用 addAll(c) 方法失败后，只有部分 c 中的元素进入队列。
* BlockingQueue 默认不支持任何形式的 close 或 shutdown 操作，来确保后续没有元素可以被添加到队列中。这个功能依赖于实现的方式。
* 内存一致性影响：和其它 concurrent Collections 一样，是线程安全的，遵循 happen-before 规则。

```java
public interface BlockingQueue<E> extends Queue<E> {
    
    boolean add(E e);

    // 有容量限制时，更倾向于使用 offer，而不是 add
    boolean offer(E e):

    void put(E e) throws InterruptedException;

    boolean offer(E e, long timeunit, TimeUnit unit)
        throws InterruptedException;

    E take() throws InterruptedException;

    E poll(long timeout, TimeUnit unit)
        throws InterruptedException;

    int remainingCapacity();

    boolean remove(Object o);

    public boolean contains(Object o);

    int drainTo(Collections<? super E> c);

    int drainTo(Collections<? super E> c, int maxElements);
}
```
