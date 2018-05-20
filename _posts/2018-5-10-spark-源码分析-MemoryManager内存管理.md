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

spark 是一个基于内存的分布式计算引擎，虽然 spark 也支持磁盘存储，但其主要的优势在于充分利用内存计算，因此理解它的内存管理原理非常重要。
另外由于它本身也对内存的使用和管理进行了一系列的优化，因为理解这些优化点，对于深入理解 spark 的内存管理模型、甚至自己开发程序都很有帮助。

内存的使用和管理，个人认为可以分成两个不同的问题：
1. 内存的管理，在于其内存模型，如何充分利用内存完成计算任务。
2. 内存的使用，则主要在于如何申请内存并优化读写，尤其是 rdd 缓存时申请内存和 task 计算时申请内存两种方式的不同。

当然，由于内存使用和管理是紧密联系的，个人只是为了写博客时能从一个尽量小的关注点切入才这样处理。因此，本文**只分析内存的管理模型**, 内存申请和使用的流程，在后面的博客中再进行分析。

由于 spark 内存管理模型，网上分析的文章已经很多，这里会引用一些网上的文章。如引用 1 讲的就比较明确，但未结合源码，讲的层次比较高，对于入门者来说略有难度。这里我尽量学习对方的优点，
并尝试结合源码来分析。

spark 内存按作用分类，主要分为存储内存(Storage Memory)、执行内存(Execution Memory)和其它内存(Other Memory).
1. 存储内存：主要用来缓存 rdd 数据、广播变量数据等；
2. 执行内存：主要是任务执行 shuffle 时占用的内存
3. 其它内存：除上述两种数据外的其它数据，一般都在其它内存部分，如 spark 内存的实例对象，用户定义的 spark 应用中的对象等。

spark 内存的存储级别有 7 个，这是由类 StorageLevel 决定的: StorageLevel 可以决定数据使用磁盘、堆内内存、堆外内存进行存储，还可以决定数据是否序列化以及是否备份。

```scala
class StorageLevel private(
    private var _useDisk: Boolean,
    private var _useMemory: Boolean,
    private var _useOffHeap: Boolean,
    private var _deserialized: Boolean,
    private var _replication: Int = 1)
```

数据的序列化带来的好处是：
1. 数据序列化后会连续存储，能充分利用内存空间，数据访问也更快。
2. 数据序列化后可以精确计算数据序列化后的大小，对内存的使用更加精确。

但数据序列化会带来 cpu 消耗，使用数据时还需要逆序列化为 java 对象，因为需要根据自己的需要决定。

从而，spark 数据存储的级别有：

```scala
/** 不存储 */
val NONE = new StorageLevel(false, false, false, false)
/** 只使用磁盘存储，只有一个副本，即无备份 */
val DISK_ONLY = new StorageLevel(true, false, false, false)
/** 只使用磁盘存储，但有2个副本，即有一个远程备份 */
val DISK_ONLY_2 = new StorageLevel(true, false, false, false, 2)
val MEMORY_ONLY = new StorageLevel(false, true, false, true)
val MEMORY_ONLY_2 = new StorageLevel(false, true, false, true, 2)
/** 只使用堆内内存存储，但存储前要进行序列化，只有一个副本，即无备份 */
val MEMORY_ONLY_SER = new StorageLevel(false, true, false, false)
/** 只使用堆内内存存储，但存储前要进行序列化，但有2个副本，即有一个远程备份 */
val MEMORY_ONLY_SER_2 = new StorageLevel(false, true, false, false, 2)
/** 同时使用堆内内存和磁盘进行存储，即当堆内内存不足时，可以写入磁盘 */
val MEMORY_AND_DISK = new StorageLevel(true, true, false, true)
val MEMORY_AND_DISK_2 = new StorageLevel(true, true, false, true, 2)
val MEMORY_AND_DISK_SER = new StorageLevel(true, true, false, false)
val MEMORY_AND_DISK_SER_2 = new StorageLevel(true, true, false, false, 2)
/** 可以使用磁盘、堆内内存、堆外内存进行存储，只有一个副本 */
val OFF_HEAP = new StorageLevel(true, true, true, false, 1)
```

## 堆内外内存

很明确，我们提到的 spark 内存，指的是 executor 端内存管理，而非 driver 端的内存。executor 端的内存，包括堆内内存与堆外内存两部分。

spark 对堆内内存的使用有着明确的规划和分配，以便充分利用内存。同时，spark 也引入了堆外内存，从而可以直接在 worker 节点的系统内存中开辟内存使用，进一步优化了内存的使用。


### 堆内内存

spark 堆内内存是在启动时，通过参数 spark.executor.memory 指定的。
由于 spark 对堆内内存的管理是一种逻辑上的“规划式”的管理，因为对象实例占用内存的申请和释放都由 JVM 完成，spark 只能在申请后和释放前记录这些内存, 如图所示：

<div class="mermaid">
graph TB
    subgraph 释放内存
    b1(Spark 记录对象释放内存的大小并删除对象引用)-->b2(等待 JVM 回收对象占用的堆内内存)
    end
    subgraph 申请内存
    a1(Spark 中 new 对象实例)-->a2(JVM 分配内存并创建对象返回引用)
    a2-->a3(Spark 保存对象引用 并记录对象占用的内存)
    end
</div>

### 堆外内存

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
堆内内存的分配, 如图所示:
![StaticMemoryManager](https://github.com/mzl9039/mzl9039.github.io/raw/master/styles/img/spark-static-memory-mode.png)

### UnifiedMemoryManager 统一内存管理

## 引用

1. [Apache Spark 内存管理详解](https://www.ibm.com/developerworks/cn/analytics/library/ba-cn-apache-spark-memory-management/index.html?ca=drs-&utm_source=tuicool&utm_medium=referral)
