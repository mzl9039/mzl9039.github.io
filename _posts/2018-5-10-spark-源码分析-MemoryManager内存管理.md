---
layout:     post
title:      Spark MemoryManager 内存管理
subtitle:   
date:       2018-05-09 00:00:00
author:     mzl
catalog:    true
tags:
    - spark
    - memoryManager
    - task
---

{:toc}

# MemoryManager 内存管理

spark 的计算模型又叫"内存计算模型"，虽然 spark 也支持磁盘存储，但其主要的优势在于充分利用内存计算。
它本身也对内存的使用和管理进行了一系列的优化。

内存的使用和管理，是两个不同的问题。内存的管理，在于其内存模型，如何充分利用内存完成计算任务。而内存使用，
则主要在于如何申请内存并优化读写，尤其是 rdd 缓存时申请内存和 task 计算时申请内存两种方式的不同。本文重点
分析内存的管理模型，内存的使用，根据需要在本文中完成，或在以后的博文中再进行分析。

由于 spark 内存管理模型，网上分析的文章已经很多，这里会引用一些网上的文章。

## MemoryManager 内存管理涉及主要的类

1. MemoryManager: 抽象的内存管理器，能加强存储内存和执行内存之间的内存共享。
2. StaticMemoryManager: 静态内存管理器，把堆内存分区为不相交的部分，即存储内存和执行内存无法共享，是 spark 1.5 及以前的版本中的内存管理器。
3. UnifiedMemoryManager: 统一内存管理器，通过一个在存储内存和执行内存之间的软边界，使双方在自身内存不足时，可以临时从对方借用内存。
4. StorageMemoryPool: 存储内存池，通过订阅机制来管理可调整大小的内存，即可以管理堆上存储内存，也可以管理堆上的执行内存，甚至可以管理堆外内存。
需要注意的是，由于是订阅机制，即使 spark 已经 release 了内存，但 jvm 可能并没有将内存释放掉，所以仍然可能造成 OOM.


## MemoryManager 内存大小初始化

从 MemoryManager 的构造函数可以看出，在初始化 MemoryManager 时就明确了堆上的存储内存 onHeapStorageMemory 和堆上的执行内存 onHeapExecutionMemory 的大小，
而堆外内存则是实例化 MemoryManager 时，从配置读取字段 `spark.memory.offHeap.size` 获取到的。

```scala
/** 初始化 MemoryManager 时，给定了堆上存储内存和堆上执行内存的大小 */
private[spark] abstract class MemoryManager(
    conf: SparkConf,
    numCores: Int,
    onHeapStorageMemory: Long,
    onHeapExecutionMemory: Long) extends Logging {

  /** -- Methods related to memory allocation policies and bookkeeping ------------------------------ */

  /** 堆上/堆外 存储内存，以及堆上/堆外 执行内存，都是 StorageMemoryPool, 即其管理都是 bookkeep 机制 */
  @GuardedBy("this")
  protected val onHeapStorageMemoryPool = new StorageMemoryPool(this, MemoryMode.ON_HEAP)
  @GuardedBy("this")
  protected val offHeapStorageMemoryPool = new StorageMemoryPool(this, MemoryMode.OFF_HEAP)
  @GuardedBy("this")
  protected val onHeapExecutionMemoryPool = new ExecutionMemoryPool(this, MemoryMode.ON_HEAP)
  @GuardedBy("this")
  protected val offHeapExecutionMemoryPool = new ExecutionMemoryPool(this, MemoryMode.OFF_HEAP)

  /** 初始化堆上存储内存/执行内存的最大值 */
  onHeapStorageMemoryPool.incrementPoolSize(onHeapStorageMemory)
  onHeapExecutionMemoryPool.incrementPoolSize(onHeapExecutionMemory)

  /** 从配置文件获取堆外内存的最大值，并进一步确认堆外存储内存和堆外执行内存的比例 */
  protected[this] val maxOffHeapMemory = conf.getSizeAsBytes("spark.memory.offHeap.size", 0)
  protected[this] val offHeapStorageMemory =
    (maxOffHeapMemory * conf.getDouble("spark.memory.storageFraction", 0.5)).toLong

  /** 初始化堆外存储内存/执行内存的最大值 */
  offHeapExecutionMemoryPool.incrementPoolSize(maxOffHeapMemory - offHeapStorageMemory)
  offHeapStorageMemoryPool.incrementPoolSize(offHeapStorageMemory)

  /** skip other codes here */
}
```

## MemoryManager 内存管理接口

MemoryManager 为管理存储内存和执行内存提供了统一的接口, 只需要指定 MemoryMode，即使用堆内内存还是堆外内存，即可完成
内存的管理操作。

```scala
/** 获取存储内存 */
def acquireStorageMemory(blockId: BlockId, numBytes: Long, memoryMode: MemoryMode): Boolean
/** 获取执行内存 */
def acquireExecutionMemory(numBytes: Long, taskAttemptId: Long, memoryMode: MemoryMode): Long
/** 获取展开内存, 展开内存是执行 shuffle 过程中，展开 rdd 的 iterator 等消耗的内存, 展开内存使用的也是存储内存 */
def acquireUnrollMemory(blockId: BlockId, numBytes: Long, memoryMode: MemoryMode): Boolean
/** 释放存储内存 */
def releaseStorageMemory(numBytes: Long, memoryMode: MemoryMode): Unit
/** 释放执行内存 */
def releaseExecutionMemory(numBytes: Long, taskAttemptId: Long, memoryMode: MemoryMode): Unit
/** 释放展开内存 */
def releaseUnrollMemory(numBytes: Long, memoryMode: MemoryMode): Unit
```

## 内存空间分配

前面提到，spark 的 MemoryManager 是一个抽象类，它的真正实现是 StaticMemoryManager 和 UnifiedMemoryManager, 通过配置
`spark.memory.useLegacyMode` 决定是否使用遗留的模式，默认为 false，即使用 UnifiedMemoryManager.

### StaticMemoryManager 静态内存管理

静态内存管理，是指存储内存、执行内存和其它内存的大小在 spark 应用程序运行期间固定不变，但用户可以在程序启动之前配置，
堆内内存的分配 ![如图2](https://github.com/mzl9039/mzl9039.github.io/tree/master/styles/img/spark-static-memory-mode.png)所示

### UnifiedMemoryManager 统一内存管理

## 引用

1. [Apache Spark 内存管理详解](https://www.ibm.com/developerworks/cn/analytics/library/ba-cn-apache-spark-memory-management/index.html?ca=drs-&utm_source=tuicool&utm_medium=referral)
