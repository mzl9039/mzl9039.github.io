---
layout:     post
title:      Spark Shuffle 原理
subtitle:   
date:       2018-04-12 00:00:00
author:     mzl
catalog:    true
tags:
    - spark
    - stage
    - shuffle
    - task
---

{:toc}

# Spark shuffle 原理

关于 shuffle 是什么，本文就不再介绍了，想了解的自己百度。

简而言之：在分布式环境下，shuffle 是造成性能差的主要原因。所以，了解 shuffle 的原理，以及了解针对 shuffle 的优化，是学习分布式计算迈不过去的槛儿。

常见的分布式计算框架，主要是 hadoop 的 Map/Reduce 和 spark。在 spark 刚推出的时期，spark 运算速度号称最多能达到 M/R 的 100 倍，既然是现在，M/R 也主要
用在日志文件处理等时效性不强的应用场景中，而 spark 却可以在时效性很强的场景中甚至流式计算中表现出很好的适应性。造成这一差异的关键原因，网上有很多介绍，
但个人对 M/R 的原理并没有深入了解过，这里无法说明，但主要的原因，应该是 spark 的 DAG 计算模式可以减少 shuffle 次数。但必须知道，
无论是 M/R 还是 spark 都一直在持续对 shuffle 这一性能杀手进行了优化。本文重要在于分析 spark 的 shuffle 原理，后续再分析 M/R 的 shuffle 原理和优化，并
比较 spark 与 M/R 之间的性能差异。

spark shuffle 的原理与优化其实是一个很复杂的逻辑，涉及了许多类以及相互转换，在阅读源码的过程中，个人并没有对这一过程完全理解，所以发现文中的错误，麻
烦发邮件到 mzl9039@163.com, 谢谢指正。

## Spark shuffle 的历史

从 spark 最早推出开始，就带有了 shuffle 机制，但在 spark 的 shuffle 机制经过了多次优化，目前spark 2.1.X 默认采用 Sort-Based shuffle, 而非早期的 
HashShuffle.

从配置项 spark.shuffle.manager 来看，spark 2.1.3-SNAPSHOT 版本中，支持 sort shuffle 和 tungsten-sort shuffle，但创建 ShuffleManager 时，两种方式
创建的 ShuffleManager 都是 SortShuffleManager，所以这两种配置并没有差别，即 spark 只支持一种 shuffle 方法：SortShuffleManager.

```scala
/** Let the user specify short names for shuffle managers */
val shortShuffleMgrNames = Map(
  "sort" -> classOf[org.apache.spark.shuffle.sort.SortShuffleManager].getName,
  "tungsten-sort" -> classOf[org.apache.spark.shuffle.sort.SortShuffleManager].getName)
val shuffleMgrName = conf.get("spark.shuffle.manager", "sort")
val shuffleMgrClass = shortShuffleMgrNames.getOrElse(shuffleMgrName.toLowerCase, shuffleMgrName)
val shuffleManager = instantiateClass[ShuffleManager](shuffleMgrClass)
```

但是，这只代表 spark shuffle 全部是基于 sort 的，并不代表其它 spark shuffle 只有单一的这一种，例如：关于 spark shuffle writer 有 UnsafeShuffleWriter、
BypassMergeSortShuffleWriter、SortShuffleWriter 三种，针对不同的情况有不同的优化实现。

## Spark shuffle 涉及的类

Spark shuffle 逻辑比较复杂，涉及的类也比较多，这里做简单的介绍：
1. ShuffleManager：shuffle system 的插件式接口。shuffleManager 在 SparkEnv 中被创建，在 driver 端和 executor 端均存在，由配置项 spark.shuffle.manager 设置；
driver 端用它来注册 shuffle, 而 executor 端通过它拉取或写入 shuffle 数据。
2. SortShuffleManager: 一个 sort-based shuffle，所有的记录都根据它们的目标 partition 的 id 排过序，然后写入到一个单独的 map output file. Reducers 通过
拉取这个文件的连续的 region 来读取自己部分的 map output. 为防止 map output data 数据超过内存限制，output 的排序过的一部分会写入磁盘，而这些磁盘上的文件
会 merge 到一个最终的 output file 中去。
3. ShuffleHandle: 一个不透明的 handle，shuffleManager 用它来传递 shuffle 信息给 task。
4. BaseShuffleHandle: 一个基本的 ShuffleHandle 的实现，仅用来获取 registerShuffle 的信息。
5. SerializedShuffleHandle: BaseShuffleHandle 的子类，用于唯一标识我们已经选择使用的 serialized shuffle.
6. BypassMergeSortShuffleHandle: BaseShuffleHandle 的子类，用于唯一标识我们已经选择使用的 bypass merge sort shuffle path.
7. ShuffleWriter: 在 map task 内获取，用于将 records 写入 shuffle system.
8. SortShuffleWriter: 对应 ShuffleHandle 为 BaseShuffleHandle 的 writer.
9. UnsafeShuffleWriter: 对应 ShuffleHandle 为 SerializedShuffleHandle 的 writer.
10. BypassMergeSortShuffleWriter: 对应 ShuffleHandle 为 BypassMergeSortShuffleHandle 的 writer. 这个类是 spark 早期的 HashShuffle 衍生而来，而早期的
HashShuffleWriter 很像。
11. ShuffleReader: 在 reduce task 内获取，用于读取 mapper 合并过的 records。
12. BlockStoreShuffleReader: 通过向其它节点的 block store 发送请求，从 shuffle 中按 range 范围 [startPartition, endPartition) 拉取和读取 partitions.
13. ShuffleBlockResolver: 这个接口的实现知道如何获取某个逻辑上的 shuffle block 的 block data. 实现可能需要使用 files 或 file segment 来封装 shuffle data.
这个主要在 BlockStore 中使用，当获取 shuffle data 时来抽象不同 shuffle 的实现。
14. IndexShuffleBlockResolver: 用于创建和维护 shuffle block 的 logic block 和 physical file location 之间的映射关系。同一个 map task 的 shuffle block 数据
被存储到一个单独的 consolidated data file(合并的数据文件) 中。而这个 data block 在这个 data file 中的偏移量则被存储在一个另外的 index file 中。使用
shuffle data 的 shuffleBlockId 和 reduce ID 作用名字，".data" 是文件后缀，".index" 是索引文件后缀.
15. ExternalSorter: 排序以及潜在地合并一些 (K,V) 类型的 key-value pair 来生成 (K,C) 类型的 key-combiner 类型的 pair. 使用 Partitioner, 首先将 key 分组
放到 partitions 中，其次可能利用常见的 Comparator 将每个 partition 中的 keys 进行排序。能输出一个单独的 partitioned file, 这个文件记录了每个 partition 的
不同 byte range， 适用于后面 fetch shuffle.
16. PartitionedAppendOnlyMap: 接口 WritablePartitionedPairCollection 的一个实现，同时是类 AppendOnlyMap 的子类。它的 key 是 (partitionId, key) 组成的元组，
它的 value 项是 combiner.
17. AppendOnlyMap: 一种开放的 hash table 的优化，仅针对只有追加的使用场景，即 key 不会被删除，但每个 key 的 value 可能会被更新。它不是用链表实现的，而是
用数组 Array 实现的，数据结构为：key1,value1,key2,value2...这样，详见引用3.

## Spark shuffle 的流程

Spark shuffle 是在任务运行中发生的。从[博客-Spark 任务分发与执行流程](https://mzl9039.github.io/2018/03/05/spark-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-%E4%BB%BB%E5%8A%A1%E5%88%86%E5%8F%91%E4%B8%8E%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B.html)
中，我们知道，当 task 在 Executor 上执行时，任务 task 是通过逆序列化得到的，同样的还包括 taskFiles/taskJars/taskProps，然后调用得到的 task 的 run 方法执行方法，
而这个 run 方法则会在调用 runTask 方法来执行。

我们知道，Task 有两类:ShuffleMapTask 和 ResultTask. 它们有一个区别：ShuffleMapTask 的 runTask 方法返回值为 MapStatus，而 ResultTask 的 runTask 返回值为
当前 Task 执行结果的类型。我们关注 shuffle,因此我们关注 ShuffleMapTask 的 runTask 方法，因为这里是 shuffle 写入开始的地方。

```scala
/** ShuffleMapTask 的 runTask 方法 */
override def runTask(context: TaskContext): MapStatus = {
  /** Deserialize the RDD using the broadcast variable. */
  val threadMXBean = ManagementFactory.getThreadMXBean
  val deserializeStartTime = System.currentTimeMillis()
  val deserializeStartCpuTime = if (threadMXBean.isCurrentThreadCpuTimeSupported) {
    threadMXBean.getCurrentThreadCpuTime
  } else 0L
  val ser = SparkEnv.get.closureSerializer.newInstance()
  val (rdd, dep) = ser.deserialize[(RDD[_], ShuffleDependency[_, _, _])](
    ByteBuffer.wrap(taskBinary.value), Thread.currentThread.getContextClassLoader)
  _executorDeserializeTime = System.currentTimeMillis() - deserializeStartTime
  _executorDeserializeCpuTime = if (threadMXBean.isCurrentThreadCpuTimeSupported) {
    threadMXBean.getCurrentThreadCpuTime - deserializeStartCpuTime
  } else 0L

  var writer: ShuffleWriter[Any, Any] = null
  try {
    /** 获取到 ShuffleManager, 当前版本中只有 SortShuffleManager */
    val manager = SparkEnv.get.shuffleManager
    /** 根据 dep.shuffleHandle 获取 ShuffleWriter, 注意 shuffleHandle 是 dep 的属性 */
    writer = manager.getWriter[Any, Any](dep.shuffleHandle, partitionId, context)
    /** 这里开始写 shuffle. shuffle 写从这里开始，但需要首先分析 rdd.iterator */
    writer.write(rdd.iterator(partition, context).asInstanceOf[Iterator[_ <: Product2[Any, Any]]])
    writer.stop(success = true).get
  } catch {
    case e: Exception =>
      try {
        if (writer != null) {
          writer.stop(success = false)
        }
      } catch {
        case e: Exception =>
          log.debug("Could not stop writer", e)
      }
      throw e
  }
}
```

### Rdd 的 iterator 方法

由于不同的 RDD 类型不同，调用顺序不同等原因，对分析有很大影响，所以这里我们举个官方的例子：LogQuery.scala.
在这个类中，rdd 分别是：
1. ShuffledRDD              --> reduceByKey
2. MapPartitionsRDD         --> map
3. ParallelCollectionRDD    --> parallelize

对于方法调用 `rdd.iterator(partition, context)`, iterator 方法如下：

```scala
/** Internal method to this RDD; will read from cache if applicable, or otherwise compute it. */
/** This should ''not'' be called by users directly, but is available for implementors of custom */
/** subclasses of RDD. */
/** 由于这几个 RDD 的 storageLevel 为 NONE, 所以这里调用的是 computeOrReadCheckpoint 方法 */
final def iterator(split: Partition, context: TaskContext): Iterator[T] = {
  if (storageLevel != StorageLevel.NONE) {
    getOrCompute(split, context)
  } else {
    computeOrReadCheckpoint(split, context)
  }
}

/** Compute an RDD partition or read it from a checkpoint if the RDD is checkpointing. */
/** 这几个 rdd 都没有做过 checkpoint，所以 isCheckpointedAndMaterialized 为 false, 调用方法 compute. */
private[spark] def computeOrReadCheckpoint(split: Partition, context: TaskContext): Iterator[T] =
{
  if (isCheckpointedAndMaterialized) {
    firstParent[T].iterator(split, context)
  } else {
    compute(split, context)
  }
}
```

从 compute 的调用顺序来看，LogQuery 的调用，是先调用 MapPartitionsRDD 的 compute，然后是 ParallelCollectionRDD 的
compute, 最后是 ShuffledRDD 的 compute, 因为 MapPartitionsRDD 和 ParallelCollectionRDD 在同一个 stage(ShuffleMapStage), 而 ShuffledRDD
则在第二个 stage(ResultStage)，先提交的是第一个 stage.

在 MapPartitionsRDD 的 compute 中，调用了 firstParent[T].iterator, 
由于 firstParent 即为 ParallelCollectionRDD, 即在这里调用了 ParallelCollectionRDD 的 iterator 方法，并在这个方法里调用了其 compute 方法。
而对于 ParallelCollectionRDD 的 iterator 方法，时面返回了一个 InterruptibleIterator，这个类其实只是一个委托，正在起作用的还是参数
`s.asInstanceOf[ParallelCollectionPartition[T]].iterator`,这里的 s 是 Partition.

```scala
/** compute 方法在 RDD.scala 中是一个钩子函数（类似于抽象类中的抽象函数），是由子类实现的函数, 因此不同的 */
/** RDD 类型实现不同, 例如： */
/** ShuffledRDD.compute */
override def compute(split: Partition, context: TaskContext): Iterator[(K, C)] = {
  val dep = dependencies.head.asInstanceOf[ShuffleDependency[K, V, C]]
  /** 注意这里的 compute 已经是 reducer task 阶段，去读取 shuffle 信息 */
  SparkEnv.get.shuffleManager.getReader(dep.shuffleHandle, split.index, split.index + 1, context)
    .read()
    .asInstanceOf[Iterator[(K, C)]]
}

/** MapPartitionsRDD.compute */
override def compute(split: Partition, context: TaskContext): Iterator[U] =
  f(context, split.index, firstParent[T].iterator(split, context))

/** ParallelCollectionRDD.compute */
override def compute(s: Partition, context: TaskContext): Iterator[T] = {
  /** s.asInstanceOf[ParallelCollectionPartition[T]].iterator 在这里，最终转为了初始数据的迭代器 values.iterator */
  /** 对 LogQuery.scala 而言，这里的 iterator 就是最初的 exampleApacheLogs 经过 RDD 切分之后某个 partition 的 iterator */
  new InterruptibleIterator(context, s.asInstanceOf[ParallelCollectionPartition[T]].iterator)
}
```

### Spark shuffle 的写入

上一节我们以 LogQuery 为例，分析了其在 ShuffleMapTask 阶段，对 rdd 的 iterator 的调用顺序。这一节我们尝试分析 ShuffleWriter 在 write 方法中
对 shuffle 进行写入的逻辑。

在上面分析 spark shuffle 流程时，我们知道 spark 是调用 writer 的 write 方法对 shuffle 进行写入的。这里我们以 SortShuffleWriter 为例进行分析。

#### SortShuffleWriter 的 write 方法

SortShuffleWriter 的 write 方法，主要是初始化了一个 ExternalSorter, 并将 rdd 的 partition 信息 insert sorter 内部的 PartitionedAppendOnlyMap 里。
如果这个 Map/Buffer 太大，则会将部分信息写入磁盘，即 spill 操作。

```scala
/** Write a bunch of records to this task's output */
override def write(records: Iterator[Product2[K, V]]): Unit = {
  sorter = if (dep.mapSideCombine) {
    require(dep.aggregator.isDefined, "Map-side combine without Aggregator specified!")
    new ExternalSorter[K, V, C](
      context, dep.aggregator, Some(dep.partitioner), dep.keyOrdering, dep.serializer)
  } else {
    /** In this case we pass neither an aggregator nor an ordering to the sorter, because we don't */
    /** care whether the keys get sorted in each partition; that will be done on the reduce side */
    /** if the operation being run is sortByKey. */
    new ExternalSorter[K, V, V](
      context, aggregator = None, Some(dep.partitioner), ordering = None, dep.serializer)
  }
  /** 将 records 代表的 rdd 的 partition 信息保存到 sorter 内部的 map/buffer */
  /** 若 sorter 内保存的 partition 信息过大，则会 spill 到磁盘 */
  sorter.insertAll(records)

  /** Don't bother including the time to open the merged output file in the shuffle write time, */
  /** because it just opens a single file, so is typically too fast to measure accurately */
  /** (see SPARK-3570). */
  val output = shuffleBlockResolver.getDataFile(dep.shuffleId, mapId)
  val tmp = Utils.tempFileWith(output)
  try {
    val blockId = ShuffleBlockId(dep.shuffleId, mapId, IndexShuffleBlockResolver.NOOP_REDUCE_ID)
    val partitionLengths = sorter.writePartitionedFile(blockId, tmp)
    shuffleBlockResolver.writeIndexFileAndCommit(dep.shuffleId, mapId, partitionLengths, tmp)
    mapStatus = MapStatus(blockManager.shuffleServerId, partitionLengths)
  } finally {
    if (tmp.exists() && !tmp.delete()) {
      logError(s"Error while deleting temp file ${tmp.getAbsolutePath}")
    }
  }
}
```

#### ExternalSorter 的 insertAll 方法

insertAll 方法会根据是否需要 combine, 把 records 加入 map/buffer 中。这里我们以 map 为例说明如何插入到
map 以及如何 spill 到磁盘。

这里的 changeValue 会根据情况判断:如果 key 第一次出现，则插入值；否则更新值。而且会自动扩展容量，每次扩展
1倍。

```scala
def insertAll(records: Iterator[Product2[K, V]]): Unit = {
  /** TODO: stop combining if we find that the reduction factor isn't high */
  val shouldCombine = aggregator.isDefined

  if (shouldCombine) {
    /** Combine values in-memory first using our AppendOnlyMap */
    val mergeValue = aggregator.get.mergeValue
    val createCombiner = aggregator.get.createCombiner
    var kv: Product2[K, V] = null
    val update = (hadValue: Boolean, oldValue: C) => {
      if (hadValue) mergeValue(oldValue, kv._2) else createCombiner(kv._2)
    }
    while (records.hasNext) {
      addElementsRead()
      kv = records.next()
      map.changeValue((getPartition(kv._1), kv._1), update)
      maybeSpillCollection(usingMap = true)
    }
  } else {
    /** Stick values into our buffer */
    while (records.hasNext) {
      addElementsRead()
      val kv = records.next()
      buffer.insert(getPartition(kv._1), kv._1, kv._2.asInstanceOf[C])
      maybeSpillCollection(usingMap = false)
    }
  }
}
```

### Spark shuffle 元数据的消息传递

### Spark shuffle 的读取

# 引用
1. [Spark2.x学习笔记：12、Shuffle机制](https://blog.csdn.net/chengyuqiang/article/details/78171094?locationNum=4&fps=1)
2. [深入理解Spark 2.1 Core （九）：迭代计算和Shuffle的原理与源码分析](https://blog.csdn.net/u011239443/article/details/54981376)
3. [大数据：Spark Shuffle（一）ShuffleWrite:Executor如何将Shuffle的结果进行归并写到数据文件中去](https://blog.csdn.net/raintungli/article/details/70807376)
4. [[Spark性能调优] 第二章：彻底解密Spark的HashShuffle](https://www.cnblogs.com/jcchoiling/p/6431969.html)

