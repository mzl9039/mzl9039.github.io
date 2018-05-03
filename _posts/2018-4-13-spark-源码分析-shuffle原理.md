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

常见的分布式计算框架，主要是 hadoop 的 Map/Reduce 和 spark。在 spark 刚推出的时期，spark 运算速度号称最多能达到 M/R 的 100 倍，即使是现在，M/R 也主要
用在日志文件处理等时效性不强的应用场景中，而 spark 却可以在时效性很强的场景中甚至流式计算中表现出很好的适应性。造成这一差异的关键原因，网上有很多介绍，
但个人对 M/R 的原理并没有深入了解过，这里无法说明，但主要的原因，应该是 spark 的 DAG 计算模式可以减少 shuffle 次数。但必须知道，
无论是 M/R 还是 spark 都一直在持续对 shuffle 这一性能杀手进行了优化。本文主要是分析 spark 的 shuffle 原理，后续再分析 M/R 的 shuffle 原理和优化，并
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
18. DiskBlockObjectWriter: 将 JVM 中的对象直接写入文件的类。这个类允许把数据追加到已经存在的 block。为了效率，它持有底层的文件通道。这个通道会保持 open 状态，
直到 DiskBlockObjectWriter 的 close() 方法被调用。为了防止出现错误(如正在写的过程中出错了), 调用者需要调用方法 revertPartialWritesAndClose 而不是 close 方法，
来自动 revert 那些未提交的 write 操作。
19. ShuffleBlockFetcherIterator: 一个获取多个 blocks 的迭代器。对于本地的 blocks, 它会从本地的 blockManager 拉取；对于远程的 blocks, 它使用 BlockTransferService
来拉取。它会创建一个 (BlockId, InputStream) 这样的 tuple 的迭代器，以保证调用者要接收到数据时像流水线一样。另外它能限制从远程拉取的速度，来保证拉取时不会
超过 maxBytesInFlight, 从而不会使用太多内存.
20. ShuffleClient: 读取 shuffle file 的接口，即可以是 Executor 端也可以是外部 service 端
21. NettyBlockTransferService: 使用 netty 来一次拉取多个 blocks 的一个 BlockTransferService, 是 ShuffleClient 的子类。

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

注意:有些类型的 RDD 在 compute 方法中，会直接返回一个 ExternalAppendOnlyMap, 这个类型和 ExternalSorter 很像，用在 Aggregator 和
CoGroupedRDD 中

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
  /** 这里如果写磁盘了，文件名则是 temp_shuffle_uuid */
  sorter.insertAll(records)

  /** Don't bother including the time to open the merged output file in the shuffle write time, */
  /** because it just opens a single file, so is typically too fast to measure accurately */
  /** (see SPARK-3570). */
  /** 这里会根据 shuffleId 和 mapId 获取原来的数据文件或新创建数据文件, 从前面 insertAll 的逻辑来看，第一次运行到这里时，是新建数据文件的 */
  /** 这里的文件名是 shuffleId_mapId_0_reduceId.data */
  /** IndexShuffleBlockResolver 中写明了：disk store 计划存储与 (map, reduce) 相关的 pair, 但在基于排序的、用于多个 reduce 的 shuffle output 会被拼凑成单个文件?  */
  /** TODO：上面是翻译错了？这么做的原因是什么呢? */
  val output = shuffleBlockResolver.getDataFile(dep.shuffleId, mapId)
  /** TODO: 这个 tempFileWith 方法要求的参数是一个 path，那 output 到底是一个路径还是一个文件呢? */
  val tmp = Utils.tempFileWith(output)
  try {
    /** 根据 shuffleId mapId 拿到 blockId, 这一步是关键，前面获取到 output 内部也是一样的方式 */
    val blockId = ShuffleBlockId(dep.shuffleId, mapId, IndexShuffleBlockResolver.NOOP_REDUCE_ID)
    /** 这里是合并写入的地方, 详细内容见后面的分析 */
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

#### Spillable 类的 maySpill 方法

在 insertAll 方法中，每次添加元素都会调用 maybeSpillCollection 方法，并根据方法的参数 usingMap 决定当前 map/buffer 的大小，
，并根据当前 map/buffer 的大小，调用 maybeSpill 方法决定是否需要 spill 到磁盘

```scala
/** Spill the current in-memory collection to disk if needed. */
/** @param usingMap whether we're using a map or buffer as our current in-memory collection */
private def maybeSpillCollection(usingMap: Boolean): Unit = {
  var estimatedSize = 0L
  /** 参数 usingMap 决定使用 map 还是使用 buffer */
  if (usingMap) {
    estimatedSize = map.estimateSize()
    /** 根据 estimatedSize 决定是否需要 spill, 使用 buffer 时也一样 */
    /** 若需要 spill，则在 spill 之后新创建一个 map/buffer */
    if (maybeSpill(map, estimatedSize)) {
      map = new PartitionedAppendOnlyMap[K, C]
    }
  } else {
    estimatedSize = buffer.estimateSize()
    if (maybeSpill(buffer, estimatedSize)) {
      buffer = new PartitionedPairBuffer[K, C]
    }
  }

  if (estimatedSize > _peakMemoryUsedBytes) {
    _peakMemoryUsedBytes = estimatedSize
  }
}
```

在上面的方法中，我们看到每次都会调用 maybeSpill 方法，在这个方法中决定是否需要 spill, 但注意 Spillable 的 spill 方法是个
抽象函数，其具体实现在 ExternalSorter 中：

```scala
/** Spills the current in-memory collection to disk if needed. Attempts to acquire more */
/** memory before spilling. */
/** @param collection collection to spill to disk */
/** @param currentMemory estimated size of the collection in bytes */
/** @return true if `collection` was spilled to disk; false otherwise */
protected def maybeSpill(collection: C, currentMemory: Long): Boolean = {
  var shouldSpill = false
  /** 每次函数进来都要检查是否需要 spill，条件是 collection 中元素个数是 32 的倍数，且当前内存大于内存阈值 */
  if (elementsRead % 32 == 0 && currentMemory >= myMemoryThreshold) {
    /** Claim up to double our current memory from the shuffle memory pool */
    /** 计算需要申请多少内存，申请完内存后应该是现在内存的 2 倍，所以要申请的内存是当前内存的 2 倍减去内存阈值 */
    val amountToRequest = 2 * currentMemory - myMemoryThreshold
    /** 计算能授权申请到的内存值, TODO：关于 Memory 相关的分析后续再分析 */
    val granted = acquireMemory(amountToRequest)
    /** 更新当前的内存阈值，若申请的内存都能拿到，则更新后内存阈值为当前内存的 2 倍； */
    /** 若申请的内存不能完全拿到, 则内存阈值已经是最大可用内存值，但这个值有可能还小于当前内存 */
    myMemoryThreshold += granted
    /** If we were granted too little memory to grow further (either tryToAcquire returned 0, */
    /** or we already had more memory than myMemoryThreshold), spill the current collection */
    /** 若增加内存阈值后，内存阈值仍小于当前内存(注意不是当前内存的 2 倍) ，则需要 spill */
    shouldSpill = currentMemory >= myMemoryThreshold
  }
  /** shouldSpill 为 true 的条件是：shouldSpill 为 true 或 当前 collection 中元素数量大于强制进行 spill 的集合元素数量阈值 */
  /** 这里其实是从内存和数量两个条件来决定是否需要 spill. */
  shouldSpill = shouldSpill || _elementsRead > numElementsForceSpillThreshold
  /** Actually spill */
  if (shouldSpill) {
    _spillCount += 1
    logSpillage(currentMemory)
    /** 这里进行 spill 操作, 即将内存中的数据落盘, 可知这里是落盘的关键 */
    spill(collection)
    /** 落盘后集合中元素数量为0,更新已经落盘的数据的大小，并释放内存 */
    _elementsRead = 0
    _memoryBytesSpilled += currentMemory
    releaseMemory()
  }
  shouldSpill
}
```

#### ExternalSorter 的 spill 方法

注意：在 ExternalAppendOnlyMap 类中也有一样的 spill 方法，因为那个类和 ExternalSorter 很像。

spill 方法如下

```scala
/** Spill our in-memory collection to a sorted file that we can merge later. */
/** We add this file into `spilledFiles` to find it later. */
/** @param collection whichever collection we're using (map or buffer) */
override protected[this] def spill(collection: WritablePartitionedPairCollection[K, C]): Unit = {
  /** 这里获取比较器 comparator, 并返回排序的可写 Partition 的迭代器 */
  val inMemoryIterator = collection.destructiveSortedWritablePartitionedIterator(comparator)
  /** 根据迭代器，将内存中的数据写到磁盘 */
  val spillFile = spillMemoryIteratorToDisk(inMemoryIterator)
  spills += spillFile
}
```

##### 下面先分析 spill 方法中排序的过程

###### ExternalSorter 的 comparator 方法

在 spill 方法的第一行， destructiveSortedWritablePartitionedIterator 方法的参数是比较器 comparator,
这是一个方法，它默认采用 ExternalSorter 的参数 ordering, 但若 ordering 为 None，则比较类型 K 的哈希值

```scala
/** A comparator for keys K that orders them within a partition to allow aggregation or sorting. */
/** Can be a partial ordering by hash code if a total ordering is not provided through by the */
/** user. (A partial ordering means that equal keys have comparator.compare(k, k) = 0, but some */
/** non-equal keys also have this, so we need to do a later pass to find truly equal keys). */
/** Note that we ignore this if no aggregator and no ordering are given. */
private val keyComparator: Comparator[K] = ordering.getOrElse(new Comparator[K] {
  override def compare(a: K, b: K): Int = {
    val h1 = if (a == null) 0 else a.hashCode()
    val h2 = if (b == null) 0 else b.hashCode()
    if (h1 < h2) -1 else if (h1 == h2) 0 else 1
  }
})

private def comparator: Option[Comparator[K]] = {
  if (ordering.isDefined || aggregator.isDefined) {
    Some(keyComparator)
  } else {
    None
  }
}
```

###### WritablePartitionedPairCollection 的 destructiveSortedWritablePartitionedIterator 方法

这个类是 map/buffer 的类型的父类，其部分抽象方法在子类中实现，这里以 map 为例说明.

```scala
/** Iterate through the data in order of partition ID and then the given comparator. This may */
/** destroy the underlying collection. */
def partitionedDestructiveSortedIterator(keyComparator: Option[Comparator[K]])
  : Iterator[((Int, K), V)]

/** Iterate through the data and write out the elements instead of returning them. Records are */
/** returned in order of their partition ID and then the given comparator. */
/** This may destroy the underlying collection. */
def destructiveSortedWritablePartitionedIterator(keyComparator: Option[Comparator[K]])
  : WritablePartitionedIterator = {
  /** partitionedDestructiveSortedIterator 是个抽象方法，见上。我们以 PartitionedAppendOnlyMap 的实现来说明 */
  val it = partitionedDestructiveSortedIterator(keyComparator)
  /** 我们看到，获取迭代器后，后面就是创建一个新的迭代器 WritablePartitionedIterator, 并实现了 writeNext 方法 */
  new WritablePartitionedIterator {
    /** 这里需要注意，cur 的类型是 Tuple2: ((Int, K), V) */
    private[this] var cur = if (it.hasNext) it.next() else null

    def writeNext(writer: DiskBlockObjectWriter): Unit = {
      writer.write(cur._1._2, cur._2)
      cur = if (it.hasNext) it.next() else null
    }

    def hasNext(): Boolean = cur != null

    def nextPartition(): Int = cur._1._1
  }
}
```

下面是 PartitionedAppendOnlyMap 的方法 partitionedDestructiveSortedIterator, 这个方法也只是调用了父类的方法
destructiveSortedIterator.

```scala
def partitionedDestructiveSortedIterator(keyComparator: Option[Comparator[K]])
  : Iterator[((Int, K), V)] = {
  /** 第一步也还是获取比较器 */
  val comparator = keyComparator.map(partitionKeyComparator).getOrElse(partitionComparator)
  /** 调用父类的方法 destructiveSortedIterator */
  destructiveSortedIterator(comparator)
}
```

###### AppendOnlyMap 的 destructiveSortedIterator 方法

这个方法的关键是两个步骤：
1. 整理数组中的数据，前数据整理到数组前面，使数据之间不存在 null, 即数据前面没有影响排序的索引
2. 通过 Sorter 实现排序, Sorter 内部通过 TimSort 对象完成对数据中数据的排序

```scala
/** Return an iterator of the map in sorted order. This provides a way to sort the map without */
/** using additional memory, at the expense of destroying the validity of the map. */
def destructiveSortedIterator(keyComparator: Comparator[K]): Iterator[(K, V)] = {
  destroyed = true
  /** Pack KV pairs into the front of the underlying array */
  /** 原来 map 中的数据，是通过 hash 得到的，可能不均匀地分散在 array 中(因为这个 map 的底层数据结构是 array) */
  /** 下面的调整，是把 array 中所有数据整理到 array 的前面，即数据中间不再有 null */
  var keyIndex, newIndex = 0
  while (keyIndex < capacity) {
    /** 若当前 keyIndx 对应的数据不为 null, 则把 keyIndx 放到 newIndex 的位置; */
    /** 若为空则跳过，这样保证 keyIndex 一定大于 newIndex, 不会存在前面的数据被覆盖的情况 */
    /** 由于用 array 实现的 map，所以数据结构里，偶数位存 key， 奇数位存 value */
    if (data(2 * keyIndex) != null) {
      data(2 * newIndex) = data(2 * keyIndex)
      data(2 * newIndex + 1) = data(2 * keyIndex + 1)
      newIndex += 1
    }
    keyIndex += 1
  }
  assert(curSize == newIndex + (if (haveNullValue) 1 else 0))

  /** KVArraySortDataFormat 确定了需要排序的数据的数据格式，这是针对这里的 AppendOnlyMap 这种用 array 实现 map 的方式专门写的 */
  /** Sorter 内部实例化了 TimSort 排序算法的实例，TODO:TimSort 算法后续再分析, 我们知道这里根据 key 对数据进行了排序 */
  /** 这里执行完成后，即完成了 data 数据按 keyComparator 的排序结果 */
  new Sorter(new KVArraySortDataFormat[K, AnyRef]).sort(data, 0, newIndex, keyComparator)

  /** 返回 map 排序结果的迭代器, 至此，排序的分析完成了, 后面是写磁盘的分析 */
  new Iterator[(K, V)] {
    var i = 0
    var nullValueReady = haveNullValue
    def hasNext: Boolean = (i < newIndex || nullValueReady)
    def next(): (K, V) = {
      if (nullValueReady) {
        nullValueReady = false
        (null.asInstanceOf[K], nullValue)
      } else {
        val item = (data(2 * i).asInstanceOf[K], data(2 * i + 1).asInstanceOf[V])
        i += 1
        item
      }
    }
  }
}
```

##### 下面是对 spill 方法中写磁盘的分析

###### ExternalSorter 的 spillMemoryIteratorToDisk 方法

这里将内存中的 map/buffer 的信息写到磁盘，是 shuffle 中真正的磁盘 IO 写操作，每次 flush 产生一个 FileSegment,
而返回值 SpilledFile 则记录了 blockId 和物理机上的文件的路径等信息，把逻辑信息和物理信息之间的映射

```scala
/** Spill contents of in-memory iterator to a temporary file on disk. */
private[this] def spillMemoryIteratorToDisk(inMemoryIterator: WritablePartitionedIterator)
    : SpilledFile = {
  /** Because these files may be read during shuffle, their compression must be controlled by */
  /** spark.shuffle.compress instead of spark.shuffle.spill.compress, so we need to use */
  /** createTempShuffleBlock here; see SPARK-3426 for more context. */
  /** 由于在 shuffle 过程中，这些文件可能正在被读取，所以他们的压缩格式必须由 spark.shuffle.compress 控制， */
  /** 所以这些文件必须通过方法 createTempShuffleBlock 创建, SPARK-3426 */
  val (blockId, file) = diskBlockManager.createTempShuffleBlock()

  /** These variables are reset after each flush */
  /** 真正需要 reset 的只有 objectsWritten，是 this, 而不是 these */
  var objectsWritten: Long = 0
  val spillMetrics: ShuffleWriteMetrics = new ShuffleWriteMetrics
  /** 获取到 writer 对象，由于初始化时已经指定了 file 等信息，所以直接像流一样写,然后循环 flush 即可 */
  /** 注意这里的 blockId 和 file 就指定了逻辑位置与物理位置的关系 */
  val writer: DiskBlockObjectWriter =
    blockManager.getDiskWriter(blockId, file, serInstance, fileBufferSize, spillMetrics)

  /** List of batch sizes (bytes) in the order they are written to disk */
  val batchSizes = new ArrayBuffer[Long]

  /** How many elements we have in each partition */
  val elementsPerPartition = new Array[Long](numPartitions)

  /** Flush the disk writer's contents to disk, and update relevant variables. */
  /** The writer is committed at the end of this process. */
  def flush(): Unit = {
    /** 每次 commitAndGet 都会返回一个 FileSegment, 后面会分析到 */
    val segment = writer.commitAndGet()
    batchSizes += segment.length
    _diskBytesSpilled += segment.length
    objectsWritten = 0
  }

  var success = false
  try {
    while (inMemoryIterator.hasNext) {
      val partitionId = inMemoryIterator.nextPartition()
      require(partitionId >= 0 && partitionId < numPartitions,
        s"partition Id: ${partitionId} should be in the range [0, ${numPartitions})")
      /** 这里把 writer 传进去，由于 inMemoryIterator 是排序后的 map/buffer 的迭代器，每个 Key-Value 对都对应一个 partition */
      /** 所以这里可以理解为每次写一个 partition */
      inMemoryIterator.writeNext(writer)
      /** 记录每个 partition 写了几次, 其实是当前 partition 被分成了多少份 */
      elementsPerPartition(partitionId) += 1
      objectsWritten += 1

      /** serializerBatchSize 为固定值 10000, 说明每写 10000 个 partition 的信息，flush 一交，即生成一个 FileSegment */
      if (objectsWritten == serializerBatchSize) {
        flush()
      }
    }
    /** whilte 循环后，可能有部分已经写了，但没达到 serializerBatchSize, 这部分也需要 flush */
    if (objectsWritten > 0) {
      flush()
    } else {
      /** 但如果 objectsWritten 为 0 的话，需要 revertPartialWritesAndClose， 下面分析这个方法和 writer 的 write 方法 */
      writer.revertPartialWritesAndClose()
    }
    success = true
  } finally {
    if (success) {
      writer.close()
    } else {
      /** This code path only happens if an exception was thrown above before we set success; */
      /** close our stuff and let the exception be thrown further */
      /** 如果在 success 置为 true 之前抛出了异常，则需要关闭连接并将文件删除 */
      writer.revertPartialWritesAndClose()
      if (file.exists()) {
        if (!file.delete()) {
          logWarning(s"Error deleting ${file}")
        }
      }
    }
  }

  /** 这个 SpilledFile 记录了逻辑上的 blockId 和物理上的 file 之间的对应关系，还有其它信息，如每个 partition 被分为了几部分, 每个 FileSegment 的大小等 */
  SpilledFile(file, blockId, batchSizes.toArray, elementsPerPartition)
}
```

###### DiskBlockObjectWriter 的相关方法

这个类的成员属性使用了缩写，为了知道这些缩写的意思，这里把成员属性也放到代码里, 这里几乎把整个类的代码都放过来了

```scala
/** The file channel, used for repositioning / truncating the file. */
/** 一些重要的成员属性 */
private var channel: FileChannel = null
private var mcs: ManualCloseOutputStream = null
private var bs: OutputStream = null
private var fos: FileOutputStream = null
private var ts: TimeTrackingOutputStream = null
private var objOut: SerializationStream = null
private var initialized = false
private var streamOpen = false
private var hasBeenClosed = false

/** Cursors used to represent positions in the file. */
/** 描述文件中当前位置的 cursor */
/** xxxxxxxxxx|----------|-----| */
/**           ^          ^     ^ */
/**           |          |    channel.position() */
/**           |        reportedPosition */
/**         committedPosition */
/** */
/** reportedPosition: Position at the time of the last update to the write metrics. */
/**                   上次更新后更新到 write metrics 的位置 */
/** committedPosition: Offset after last committed write. */
/**                    已经提交 write 的 offset */
/** -----: Current writes to the underlying file. */
/** xxxxx: Committed contents of the file. */
private var committedPosition = file.length()
private var reportedPosition = committedPosition

/** Keep track of number of records written and also use this to periodically */
/** output bytes written since the latter is expensive to do for each record. */
private var numRecordsWritten = 0

private def initialize(): Unit = {
  fos = new FileOutputStream(file, true)
  channel = fos.getChannel()
  ts = new TimeTrackingOutputStream(writeMetrics, fos)
  class ManualCloseBufferedOutputStream
    extends BufferedOutputStream(ts, bufferSize) with ManualCloseOutputStream
  mcs = new ManualCloseBufferedOutputStream
}

/** 只能 open 一次，且不能 reopen */
def open(): DiskBlockObjectWriter = {
  if (hasBeenClosed) {
    throw new IllegalStateException("Writer already closed. Cannot be reopened.")
  }
  if (!initialized) {
    initialize()
    initialized = true
  }

  bs = serializerManager.wrapStream(blockId, mcs)
  objOut = serializerInstance.serializeStream(bs)
  streamOpen = true
  this
}

/** Flush the partial writes and commit them as a single atomic block. */
/** A commit may write additional bytes to frame the atomic block. */
/** 指部分写入的内容提交到一个单独的原子 block */
/** @return file segment with previous offset and length committed on this call. */
/** 每次调用都会返回一个 FileSegment */
def commitAndGet(): FileSegment = {
  if (streamOpen) {
    /** NOTE: Because Kryo doesn't flush the underlying stream we explicitly flush both the */
    /**       serializer stream and the lower level stream. */
    objOut.flush()
    bs.flush()
    objOut.close()
    streamOpen = false

    /** syncWrites 黑夜为 false */
    if (syncWrites) {
      /** Force outstanding writes to disk and track how long it takes */
      /** 强制将写入的内容落到磁盘，并记录落盘的时间长短 */
      val start = System.nanoTime()
      fos.getFD.sync()
      writeMetrics.incWriteTime(System.nanoTime() - start)
    }

    val pos = channel.position()
    val fileSegment = new FileSegment(file, committedPosition, pos - committedPosition)
    committedPosition = pos
    /** In certain compression codecs, more bytes are written after streams are closed */
    writeMetrics.incBytesWritten(committedPosition - reportedPosition)
    reportedPosition = committedPosition
    fileSegment
  } else {
    new FileSegment(file, committedPosition, 0)
  }
}


/** Reverts writes that haven't been committed yet. Callers should invoke this function */
/** when there are runtime exceptions. This method will not throw, though it may be */
/** unsuccessful in truncating written data. */
/** revert 那些还没提交的写入内容。 */
/** @return the file that this DiskBlockObjectWriter wrote to. */
def revertPartialWritesAndClose(): File = {
  /** Discard current writes. We do this by flushing the outstanding writes and then */
  /** truncating the file to its initial position. */
  Utils.tryWithSafeFinally {
    if (initialized) {
      writeMetrics.decBytesWritten(reportedPosition - committedPosition)
      writeMetrics.decRecordsWritten(numRecordsWritten)
      streamOpen = false
      closeResources()
    }
  } {
    var truncateStream: FileOutputStream = null
    try {
      truncateStream = new FileOutputStream(file, true)
      truncateStream.getChannel.truncate(committedPosition)
    } catch {
      case e: Exception =>
        logError("Uncaught exception while reverting partial writes to file " + file, e)
    } finally {
      if (truncateStream != null) {
        truncateStream.close()
        truncateStream = null
      }
    }
  }
  file
}

/** Writes a key-value pair. */
def write(key: Any, value: Any) {
  if (!streamOpen) {
    open()
  }

  objOut.writeKey(key)
  objOut.writeValue(value)
  recordWritten()
}

override def write(b: Int): Unit = throw new UnsupportedOperationException()

override def write(kvBytes: Array[Byte], offs: Int, len: Int): Unit = {
  if (!streamOpen) {
    open()
  }

  bs.write(kvBytes, offs, len)
}

/** Notify the writer that a record worth of bytes has been written with OutputStream#write. */
/** 通知 writer: 一条记录已经被写入 OutputStream */
def recordWritten(): Unit = {
  /** 每次都会记录一次 numRecordsWritten */
  numRecordsWritten += 1
  writeMetrics.incRecordsWritten(1)

  if (numRecordsWritten % 16384 == 0) {
    updateBytesWritten()
  }
}

/** Report the number of bytes written in this writer's shuffle write metrics. */
/** Note that this is only valid before the underlying streams are closed. */
private def updateBytesWritten() {
  val pos = channel.position()
  writeMetrics.incBytesWritten(pos - reportedPosition)
  reportedPosition = pos
}
```

##### ExternalSorter 的 writePartitionedFile 方法

这里方法是把 ExternalSorter 里已经记录的所有数据写入到一个文件，注意的是，如果这些数据没有排序过，会先排序; 另外如果有部分数据因为内存不够而已经写到磁盘，
会把这些文件先合并

```scala
/** Write all the data added into this ExternalSorter into a file in the disk store. This is */
/** called by the SortShuffleWriter. */
/** 将所有加入 ExternalSorter 的数据写入一个文件。 */
/** @param blockId block ID to write to. The index file will be blockId.name + ".index". */
/** @return array of lengths, in bytes, of each partition of the file (used by map output tracker) */
/** 返回值是一个数组，数组的每一项都是这个方法中输出的文件中某个 partition 的长度 (in bytes), 后面被 map output tracker 使用 */
def writePartitionedFile(
    blockId: BlockId,
    outputFile: File): Array[Long] = {

  /** Track location of each range in the output file */
  /** 追踪文件中每个 partition 的 length */
  val lengths = new Array[Long](numPartitions)
  val writer = blockManager.getDiskWriter(blockId, outputFile, serInstance, fileBufferSize,
    context.taskMetrics().shuffleWriteMetrics)

  /** 如果原来的 map/buffer 已经有一总分写磁盘了，则 spills 不为空，否则为空 */
  if (spills.isEmpty) {
    /** Case where we only have in-memory data */
    /** 如果内存够大，所有数据都在内存中，没 spill 到磁盘过 */
    val collection = if (aggregator.isDefined) map else buffer
    /** 获取加入 ExternalSorter 的数据的集合的迭代器, 这个方法我们前面已经分析过 */
    val it = collection.destructiveSortedWritablePartitionedIterator(comparator)
    while (it.hasNext) {
      val partitionId = it.nextPartition()
      /** while 里的判断，可能是为了避免多线程时，前后句之间有线程执行了 it.next */
      while (it.hasNext && it.nextPartition() == partitionId) {
        it.writeNext(writer)
      }
      /** 这里可以看出，每个 partition 提交一次, 注意这里没有 flush */
      val segment = writer.commitAndGet()
      lengths(partitionId) = segment.length
    }
  } else {
    /** We must perform merge-sort; get an iterator by partition and write everything directly. */
    /** 如果已经有部分数据写磁盘了 */
    for ((id, elements) <- this.partitionedIterator) {
      if (elements.hasNext) {
        for (elem <- elements) {
          writer.write(elem._1, elem._2)
        }
        val segment = writer.commitAndGet()
        lengths(id) = segment.length
      }
    }
  }

  writer.close()
  context.taskMetrics().incMemoryBytesSpilled(memoryBytesSpilled)
  context.taskMetrics().incDiskBytesSpilled(diskBytesSpilled)
  context.taskMetrics().incPeakExecutionMemory(peakMemoryUsedBytes)

  lengths
}
```

###### ExternalSorter 的 partitionedIterator 方法

这个方法最终就是返回当前对象中的所有数据的迭代器，这个迭代器以 partition 为单位

```scala
/** Return an iterator over all the data written to this object, grouped by partition and */
/** aggregated by the requested aggregator. For each partition we then have an iterator over its */
/** contents, and these are expected to be accessed in order (you can't "skip ahead" to one */
/** partition without reading the previous one). Guaranteed to return a key-value pair for each */
/** partition, in order of partition ID. */
/** 返回所有写入这个对象的数据的迭代器，按 partition 分组，并根据请求的 aggregator 进行聚合。对每个 partition, 我们都有一个迭代器来遍历它， */
/** 这些 partition 将会被按顺序访问，不能跳跃。保证返回一个 key-value pair，安排好 partition ID 排序 */
/** For now, we just merge all the spilled files in once pass, but this can be modified to */
/** support hierarchical merging. */
/** Exposed for testing. */
def partitionedIterator: Iterator[(Int, Iterator[Product2[K, C]])] = {
  /** 这里只在决定当前对象使用的是 map 还是 buffer */
  val usingMap = aggregator.isDefined
  val collection: WritablePartitionedPairCollection[K, C] = if (usingMap) map else buffer
  /** 若 spills 为 empty, 说明之前没有数据被  spill 到磁盘; 否则有数据被 spill 到磁盘 */
  if (spills.isEmpty) {
    /** Special case: if we have only in-memory data, we don't need to merge streams, and perhaps */
    /** we don't even need to sort by anything other than partition ID */
    /** 特殊情况：如果我们只有内存中的数据，我们不需要把不同的 streams merge 到一起，甚至我们除了 partition ID 外，不需要按其它任何东西排序 */
    /** 如果 ordering 定义过，除了需要按照 partition ID 排序外，还需要按照 key 排序；否则只需要按照 partition ID 排序 */
    if (!ordering.isDefined) {
      /** The user hasn't requested sorted keys, so only sort by partition ID, not key */
      groupByPartition(destructiveIterator(collection.partitionedDestructiveSortedIterator(None)))
    } else {
      /** We do need to sort by both partition ID and key */
      groupByPartition(destructiveIterator(
        collection.partitionedDestructiveSortedIterator(Some(keyComparator))))
    }
  } else {
    /** Merge spilled and in-memory data */
    /** 当已经有数据 spill 到磁盘，则把 spill 到磁盘上的数据，和内存中的数据 merge 到一起 */
    merge(spills, destructiveIterator(
      collection.partitionedDestructiveSortedIterator(comparator)))
  }
}
```

###### ExternalSorter 的 groupByPartition 方法

方法 groupByPartition 写的很明确，参数是一个迭代器，里面每个项都是 ((partition, key), combiner) 这样的类型，而且这些数据
已经按照 partition ID 完成排序. 这里要把这些数据里的每个 partition 组合为 (partition, (key, combiner)) 这样的类型

```scala
/** Given a stream of ((partition, key), combiner) pairs *assumed to be sorted by partition ID*, */
/** group together the pairs for each partition into a sub-iterator. */
/** @param data an iterator of elements, assumed to already be sorted by partition ID */
private def groupByPartition(data: Iterator[((Int, K), C)])
    : Iterator[(Int, Iterator[Product2[K, C]])] =
{
  val buffered = data.buffered
  (0 until numPartitions).iterator.map(p => (p, new IteratorForPartition(p, buffered)))
}

/** An iterator that reads only the elements for a given partition ID from an underlying buffered */
/** stream, assuming this partition is the next one to be read. Used to make it easier to return */
/** partitioned iterators from our in-memory collection. */
private[this] class IteratorForPartition(partitionId: Int, data: BufferedIterator[((Int, K), C)])
  extends Iterator[Product2[K, C]]
{
  override def hasNext: Boolean = data.hasNext && data.head._1._1 == partitionId

  override def next(): Product2[K, C] = {
    if (!hasNext) {
      throw new NoSuchElementException
    }
    val elem = data.next()
    (elem._1._2, elem._2)
  }
}
```

###### ExternalSorter 的 merge 方法

在 partitionedIterator 中，若 spills.isEmpty 为 false, 即已经有部分数据已经写入到磁盘，则需要调用 merge 方法，
将还在内存中的数据和磁盘上的数据进行 merge 操作，最终返回一个按 partition 分组的，对所有写入当前对象的数据
都可以访问的迭代器（注意这些数据可以同时在磁盘和内存中）

```scala
/** Merge a sequence of sorted files, giving an iterator over partitions and then over elements */
/** inside each partition. This can be used to either write out a new file or return data to */
/** the user. */
/** */
/** Returns an iterator over all the data written to this object, grouped by partition. For each */
/** partition we then have an iterator over its contents, and these are expected to be accessed */
/** in order (you can't "skip ahead" to one partition without reading the previous one). */
/** Guaranteed to return a key-value pair for each partition, in order of partition ID. */
private def merge(spills: Seq[SpilledFile], inMemory: Iterator[((Int, K), C)])
    : Iterator[(Int, Iterator[Product2[K, C]])] = {
  /** 磁盘上的文件中的数据的读取访问，通过 SpillReader 完成, 所以这里每个文件生成一个 reader */
  val readers = spills.map(new SpillReader(_))
  val inMemBuffered = inMemory.buffered
  /** 以 partition 来分组，所以按 partition 的个数来 iterator */
  (0 until numPartitions).iterator.map { p =>
    /** 注意这里 p 是 partition 的 id， 即 partitionId */
    /** 内存中的数据的遍历，通过 IteratorForPartition 完成,这里生成内存中数据访问的迭代器 */
    val inMemIterator = new IteratorForPartition(p, inMemBuffered)
    /** 文件中的数据的访问，通过 reader 的方法 readNextPartition 生成迭代器，和内存中的数据的迭代器一起，形成最终的迭代器 */
    val iterators = readers.map(_.readNextPartition()) ++ Seq(inMemIterator)
    /** 如果定义了聚合，则需要 mergeWithAggregation */
    /** TODO:这里不会同时定义，对么? 或者说，只需要其中一个就可以了，比如聚合定义了，排序就没有意义了? */
    if (aggregator.isDefined) {
      /** Perform partial aggregation across partitions */
      (p, mergeWithAggregation(
        iterators, aggregator.get.mergeCombiners, keyComparator, ordering.isDefined))
    } else if (ordering.isDefined) {
      /** No aggregator given, but we have an ordering (e.g. used by reduce tasks in sortByKey); */
      /** sort the elements without trying to merge them */
      /** 如果定义了排序，则需要 mergeSort */
      (p, mergeSort(iterators, ordering.get))
    } else {
      (p, iterators.iterator.flatten)
    }
  }
}
```

###### Spillable 的 readNextPartition 方法

这里返回当前 Spillable 对应的 SpilledFile 中所有的 partition 的 (key, combiner) 的迭代器

```scala
/** 由这个方法看出，这里尝试读取下一个 partition, 而方法的关键，在于 readNextItem, 这里返回的是下一个 partition 的 (k, c) pair */
var nextPartitionToRead = 0

def readNextPartition(): Iterator[Product2[K, C]] = new Iterator[Product2[K, C]] {
  val myPartition = nextPartitionToRead
  nextPartitionToRead += 1

  override def hasNext: Boolean = {
    if (nextItem == null) {
      nextItem = readNextItem()
      if (nextItem == null) {
        return false
      }
    }
    assert(lastPartitionId >= myPartition)
    /** Check that we're still in the right partition; note that readNextItem will have returned */
    /** null at EOF above so we would've returned false there */
    lastPartitionId == myPartition
  }

  override def next(): Product2[K, C] = {
    if (!hasNext) {
      throw new NoSuchElementException
    }
    val item = nextItem
    nextItem = null
    item
  }
}

/** Return the next (K, C) pair from the deserialization stream and update partitionId, */
/** indexInPartition, indexInBatch and such to match its location. */
/** 由于数据 spill 到磁盘上的时候，每个 SpilledFile 文件记录了这个 SpilledFile 文件的大小，及其在文件中的 offset(以 byte 为单位)， */
/** 因为写磁盘的时候，每 flush 一次，会将先前的写入提交一次，从而生成一个 FileSegment，这个 FileSegment 记录了这次提交的数据量的大小(以 byte 为单位) */
/** 对应到物理机上，其实多个 SpilledFile 是同一个文件；所以可以根据 offset, 很容易地定义到需要获取的文件流的起始位置与结束位置, */
/** 这是 nextBatchStream 这个方法的底层原理 */
/** SpilledFile 的属性 elementsPerPartition 是同一个 SpilledFile 中，相同的 partition 被访问了几次，注意这里相同的 partition 可能进同一个 SpilledFile */
/** If the current batch is drained, construct a stream for the next batch and read from it. */
/** If no more pairs are left, return null. */
/** 这里从磁盘文件中读取一个 stream, 这个 stream 对应一个 batch, 一个 batch 在文件中对应一个 FileSegment, 由于一个 FileSegment 有多个 partition */
/** 这里在 indexInBatch 等于 serializerBatchSize 时，才读取下一个 batch, 否则一直在当前的 batch stream 中读取下一个 partition */
private def readNextItem(): (K, C) = {
  if (finished || deserializeStream == null) {
    return null
  }
  /** 从 stream 中读取一个 partition 的 key 和 combiner */
  val k = deserializeStream.readKey().asInstanceOf[K]
  val c = deserializeStream.readValue().asInstanceOf[C]
  lastPartitionId = partitionId
  /** Start reading the next batch if we're done with this one */
  indexInBatch += 1
  if (indexInBatch == serializerBatchSize) {
    indexInBatch = 0
    deserializeStream = nextBatchStream()
  }
  /** Update the partition location of the element we're reading */
  indexInPartition += 1
  skipToNextPartition()
  /** If we've finished reading the last partition, remember that we're done */
  if (partitionId == numPartitions) {
    finished = true
    if (deserializeStream != null) {
      deserializeStream.close()
    }
  }
  (k, c)
}

/** Construct a stream that only reads from the next batch */
def nextBatchStream(): DeserializationStream = {
  /** Note that batchOffsets.length = numBatches + 1 since we did a scan above; check whether */
  /** we're still in a valid batch. */
  /** 由于上面调用 scanLeft(0)(_ + _), 所以 batchOffsets 要比 numBatches 大 1, 所以这里检查当前是否是个有效的 batch */
  if (batchId < batchOffsets.length - 1) {
    if (deserializeStream != null) {
      deserializeStream.close()
      fileStream.close()
      deserializeStream = null
      fileStream = null
    }

    /** 由于 batchOffsets 是由不同的 batch 的 size 这个数组逐渐累加的(scanLeft(0L)(_ + _))，类似于斐波那契数列一样, */
    /** 所以根据 batchId 即可前面多个 batch size 相加后的和，即当前 batchId 的起始 offset */
    val start = batchOffsets(batchId)
    fileStream = new FileInputStream(spill.file)
    /** 由于拿到了当前 batchId 的 start，因此能一次性定义到位置 */
    fileStream.getChannel.position(start)
    batchId += 1

    val end = batchOffsets(batchId)

    assert(end >= start, "start = " + start + ", end = " + end +
      ", batchOffsets = " + batchOffsets.mkString("[", ", ", "]"))

    val bufferedStream = new BufferedInputStream(ByteStreams.limit(fileStream, end - start))

    val wrappedStream = serializerManager.wrapStream(spill.blockId, bufferedStream)
    serInstance.deserializeStream(wrappedStream)
  } else {
    /** No more batches left */
    cleanup()
    null
  }
}

/** Update partitionId if we have reached the end of our current partition, possibly skipping */
/** empty partitions on the way. */
/** 这个方法用在跳过空的 partition(如在 Spillable 初始化时调用过)时，以及用在到当前 partition 的尾部时 */
/** 更新并记录当前对象的 partitionId 和 indexInPartition 信息, 以便后续使用 */
private def skipToNextPartition() {
  while (partitionId < numPartitions &&
      indexInPartition == spill.elementsPerPartition(partitionId)) {
    partitionId += 1
    indexInPartition = 0L
  }
}
```

###### ExternalSorter 的 mergeWithAggregation 方法

这个方法主要是对结果进行聚合，即根据参数 mergeCombiners 对相同 key 的 partition 执行 combiner 操作。
由于结果可能已经按 key 排序过，所以要区分是否已经 totalOrder.

由方法可知,这里返回的结果，都是通过 mergeSort 进行排序后的结果, 所以 mergeSort 方法决定了 next 的顺序

```scala
/** Merge a sequence of (K, C) iterators by aggregating values for each key, assuming that each */
/** iterator is sorted by key with a given comparator. If the comparator is not a total ordering */
/** (e.g. when we sort objects by hash code and different keys may compare as equal although */
/** they're not), we still merge them by doing equality tests for all keys that compare as equal. */
/** 根据要聚合的值，为每个 key (对应一个 partition),将一个序列的 iterator 聚合到一起. 假定每个 */
/** iterator 按给定的 comparator 对 key 进行排序 */
private def mergeWithAggregation(
    iterators: Seq[Iterator[Product2[K, C]]],
    mergeCombiners: (C, C) => C,
    comparator: Comparator[K],
    totalOrder: Boolean)
    : Iterator[Product2[K, C]] =
{
  /** totalOrder: orderging 是否定义过 */
  if (!totalOrder) {
    /** We only have a partial ordering, e.g. comparing the keys by hash code, which means that */
    /** multiple distinct keys might be treated as equal by the ordering. To deal with this, we */
    /** need to read all keys considered equal by the ordering at once and compare them. */
    new Iterator[Iterator[Product2[K, C]]] {
      /** 初始化时要根据 comparator 对 iterator 中的 key 进行排序 */
      val sorted = mergeSort(iterators, comparator).buffered

      /** Buffers reused across elements to decrease memory allocation */
      /** 这里使用 ArrayBuffer 是为了减少内存分配 */
      /** 其中 keys 用来存储 iterator 中的 key, combiners 用来存储其中的 combiner */
      val keys = new ArrayBuffer[K]
      val combiners = new ArrayBuffer[C]

      override def hasNext: Boolean = sorted.hasNext

      override def next(): Iterator[Product2[K, C]] = {
        if (!hasNext) {
          throw new NoSuchElementException
        }
        /** 为后面的 merge 做准备 */
        keys.clear()
        combiners.clear()
        /** 获取第一个 pair */
        val firstPair = sorted.next()
        keys += firstPair._1
        combiners += firstPair._2
        val key = firstPair._1
        /** 这里遍历 iterator，获取所有的 (K, C) */
        while (sorted.hasNext && comparator.compare(sorted.head._1, key) == 0) {
          /** 如果有下一个 pair,则获取下一个 pair */
          val pair = sorted.next()
          var i = 0
          var foundKey = false
          /** 对下一个 pair, 从 i=0 开始遍历 keys, 如果能找到和 pair 相同的 key, 则 merge, 否则继续遍历 */
          while (i < keys.size && !foundKey) {
            /** 如果 pair 的 key 与 keys(i) 相同，则进行 merge, 则设置 foundKey 为true, 即不再循环; 否则继续遍历 */
            if (keys(i) == pair._1) {
              /** 这里的 mergeCombiners 是参数，也是一个方法，这里是调用方法完成对相同 key 的 merge */
              combiners(i) = mergeCombiners(combiners(i), pair._2)
              foundKey = true
            }
            i += 1
          }
          /** 如果遍历 keys 都没有找到相同的 key, 则添加到 keys 和 combiners 中去 */
          if (!foundKey) {
            keys += pair._1
            combiners += pair._2
          }
        }

        /** Note that we return an iterator of elements since we could've had many keys marked */
        /** equal by the partial order; we flatten this below to get a flat iterator of (K, C). */
        keys.iterator.zip(combiners.iterator)
      }
    }.flatMap(i => i)
  } else {
    /** We have a total ordering, so the objects with the same key are sequential. */
    /** 如果 totalOrder 为 True, 即已经排过序，相同的 key 已经是一个序列的了，则直接根据 comparator 对 partition 进行排序即可 */
    new Iterator[Product2[K, C]] {
      val sorted = mergeSort(iterators, comparator).buffered

      override def hasNext: Boolean = sorted.hasNext

      /** 这个方法的逻辑与 totalOrder 为 false 时很类似，但要简单一些，在此跳过 */
      override def next(): Product2[K, C] = {
        if (!hasNext) {
          throw new NoSuchElementException
        }
        val elem = sorted.next()
        val k = elem._1
        var c = elem._2
        while (sorted.hasNext && sorted.head._1 == k) {
          val pair = sorted.next()
          c = mergeCombiners(c, pair._2)
        }
        (k, c)
      }
    }
  }
}
```

###### ExternalSorter 的 mergeSort 方法

这个方法主要是根据 comparator 按 key 对 iterator 进行归并排序

```scala
/** Merge-sort a sequence of (K, C) iterators using a given a comparator for the keys. */
private def mergeSort(iterators: Seq[Iterator[Product2[K, C]]], comparator: Comparator[K])
    : Iterator[Product2[K, C]] =
{
  val bufferedIters = iterators.filter(_.hasNext).map(_.buffered)
  type Iter = BufferedIterator[Product2[K, C]]
  val heap = new mutable.PriorityQueue[Iter]()(new Ordering[Iter] {
    /** Use the reverse of comparator.compare because PriorityQueue dequeues the max */
    override def compare(x: Iter, y: Iter): Int = -comparator.compare(x.head._1, y.head._1)
  })
  heap.enqueue(bufferedIters: _*)  /** Will contain only the iterators with hasNext = true */
  new Iterator[Product2[K, C]] {
    override def hasNext: Boolean = !heap.isEmpty

    override def next(): Product2[K, C] = {
      if (!hasNext) {
        throw new NoSuchElementException
      }
      val firstBuf = heap.dequeue()
      val firstPair = firstBuf.next()
      if (firstBuf.hasNext) {
        heap.enqueue(firstBuf)
      }
      firstPair
    }
  }
}
```

### Spark shuffle 的读取

我们在 RDD 的 iterator 方法中，已经介绍了，对于 ShuffledRDD 的 iterator 方法，是在 ResultTask 的 runTask 中触发的，该方法
这里不再介绍，但 ShuffledRDD 的 iterator 方法，会获取 ShuffleReader 的一个实例，并调用其 read 方法来读取已经 combine 过的
key-value 数据。compute 方法在 RDD 的 iterator 方法中已经有介绍，这里继续分析 ShuffleReader 的 read 方法。

#### BlockStoreShuffleReader 的 read 方法

当前的 spark 版本中，ShuffleReader 只有一个版本的实现: BlockStoreShuffleReader. 同时，由于数据的 shuffle 和 combine 在
shuffle 写入时已经完成，所以 shuffleReader 看起来并没有多少优化的空间，只需要将 combine 过后的数据拉到 reduce 执行的节点
进行最后的结果计算，即 ResultTask 的 runTask 最后一行:`func(context, rdd.iterator(partition, context))`, 在这里，rdd.iterator
会调用 rdd 的 compute 方法，由于当前的 rdd 是 ShuffledRDD(LogQuery 的 reduceByKey), 所以在其 compute 方法中会实例化这个
BlockStoreShuffleReader 来获得 shuffleReader

```scala
/** Read the combined key-values for this reduce task */
/** 为当前的 reduce task 读取已经在 map task 中 combine 过的 key-value 值 */
override def read(): Iterator[Product2[K, C]] = {
  /** 由于 map task 中的结果存储在 block 中，这里返回拉取 block 的迭代器, 以读取 map 端的结果 */
  /** 关于这个类，后面详细分析 */
  val blockFetcherItr = new ShuffleBlockFetcherIterator(
    context,
    blockManager.shuffleClient,
    blockManager,
    /** 我们知道 shuffle 写入完成后，返回的是 MapStatus, 而 mapOutputTracker 就是用来追踪 mapStatus 的 */
    mapOutputTracker.getMapSizesByExecutorId(handle.shuffleId, startPartition, endPartition),
    /** Note: we use getSizeAsMb when no suffix is provided for backwards compatibility */
    SparkEnv.get.conf.getSizeAsMb("spark.reducer.maxSizeInFlight", "48m") * 1024 * 1024,
    SparkEnv.get.conf.getInt("spark.reducer.maxReqsInFlight", Int.MaxValue))

  /** Wrap the streams for compression and encryption based on configuration */
  /** 如果需要压缩或数据加密需求，则在这里将输入流添加一层 wrap, 思想类似于装饰者模式 */
  val wrappedStreams = blockFetcherItr.map { case (blockId, inputStream) =>
    serializerManager.wrapStream(blockId, inputStream)
  }

  val serializerInstance = dep.serializer.newInstance()

  /** Create a key/value iterator for each stream */
  /** 将输入流 wrappedStreams 逆序列化，map 成 key/value 形式的迭代器 */
  val recordIter = wrappedStreams.flatMap { wrappedStream =>
    /** Note: the asKeyValueIterator below wraps a key/value iterator inside of a */
    /** NextIterator. The NextIterator makes sure that close() is called on the */
    /** underlying InputStream when all records have been read. */
    /** 由于前面的 wrappedStream 是流，所以读取完成需要关闭，这里 asKeyValueIterator 会在读取完成后关闭流 */
    serializerInstance.deserializeStream(wrappedStream).asKeyValueIterator
  }

  /** Update the context task metrics for each record read. */
  /** 这里定义了 metricIter, 其实是要在 recordIter 读取完成后自动调用 mergeShuffleReadMetrics 方法, 理解成是对 recordIter 的一种包装 */
  val readMetrics = context.taskMetrics.createTempShuffleReadMetrics()
  val metricIter = CompletionIterator[(Any, Any), Iterator[(Any, Any)]](
    recordIter.map { record =>
      readMetrics.incRecordsRead(1)
      record
    },
    context.taskMetrics().mergeShuffleReadMetrics())

  /** An interruptible iterator must be used here in order to support task cancellation */
  /** 又对 metricIter 加了一层包装，支持了 interrupt */
  val interruptibleIter = new InterruptibleIterator[(Any, Any)](context, metricIter)

  /** 如果定义了聚合函数，则根据需要进行聚合；否则直接 asInstanceOf 即可 */
  val aggregatedIter: Iterator[Product2[K, C]] = if (dep.aggregator.isDefined) {
    /** 根据 map 端已经 combine, 将 interruptibleIter 转化为不同的类型，然后进行 reduce 端的 combine */
    if (dep.mapSideCombine) {
      /** We are reading values that are already combined */
      /** 读取已经 combine 过的结果, 然后再对 combiners 进行 combine, 注意这里是对 combiner 进行 combine, 不是对 values */
      val combinedKeyValuesIterator = interruptibleIter.asInstanceOf[Iterator[(K, C)]]
      dep.aggregator.get.combineCombinersByKey(combinedKeyValuesIterator, context)
    } else {
      /** We don't know the value type, but also don't care -- the dependency *should* */
      /** have made sure its compatible w/ this aggregator, which will convert the value */
      /** type to the combined type C */
      /** 如果 map 端没有 combine 过，则需要对 values 进行 combine */
      val keyValuesIterator = interruptibleIter.asInstanceOf[Iterator[(K, Nothing)]]
      dep.aggregator.get.combineValuesByKey(keyValuesIterator, context)
    }
  } else {
    require(!dep.mapSideCombine, "Map-side combine without Aggregator specified!")
    interruptibleIter.asInstanceOf[Iterator[Product2[K, C]]]
  }

  /** Sort the output if there is a sort ordering defined. */
  /** 如果需要对结果进行排序，则使用 ExternalSorter 进行排序，由于前面经过了 combine, map 端之前可能的排序已被打乱了 */
  dep.keyOrdering match {
    case Some(keyOrd: Ordering[K]) =>
      /** Create an ExternalSorter to sort the data. Note that if spark.shuffle.spill is disabled, */
      /** the ExternalSorter won't spill to disk. */
      /** 这里使用 ExternalSorter 对数据进行排序，前面对 ExternalSorter 有过较为详细地分析，这里的排序可能会 spill 到磁盘 */
      val sorter =
        new ExternalSorter[K, C, C](context, ordering = Some(keyOrd), serializer = dep.serializer)
      sorter.insertAll(aggregatedIter)
      /** spark 会对这个外部排序的过程记录使用的内在/磁盘大小，所以只要能获取到 metrics, 就知道这个过程占用多大的空间 */
      context.taskMetrics().incMemoryBytesSpilled(sorter.memoryBytesSpilled)
      context.taskMetrics().incDiskBytesSpilled(sorter.diskBytesSpilled)
      context.taskMetrics().incPeakExecutionMemory(sorter.peakMemoryUsedBytes)
      CompletionIterator[Product2[K, C], Iterator[Product2[K, C]]](sorter.iterator, sorter.stop())
    case None =>
      aggregatedIter
  }
}
```

#### ShuffleBlockFetcherIterator 的 next 方法

ShuffleBlockFetcherIterator 类本身是一个迭代器，用来一次拉取一些 blocks. 它不只能一些拉取多个 blocks, 还会限制拉取 blocks
的最大值，从而保证拉取的 block 不会占用大量内存，即起到加速的效果，又有限制作用。这个类重要的方法主要有初始化方法 initialize
和读取下一批 block 的 next 方法

```scala
private[this] def initialize(): Unit = {
  /** Add a task completion callback (called in both success case and failure case) to cleanup. */
  /** 添加任务的监听事件，确保释放所有的 buffer, 不论是否成功地获取到了结果，都会释放 */
  context.addTaskCompletionListener(_ => cleanup())

  /** Split local and remote blocks. */
  /** 这里会区别要拉取的 block 是本地还是远程，本地的通过本地的 blockManager 去拉； */
  /** 远程的 block, 根据 block 的 size，确保每个请求要拉取的 blocks 的 size 总和超过 targetRequestSize */
  /** 这里根据 size 大小相加，当 size 大小超过 targetRequestSize 时，封装成一个请求 */
  /** 这就保证了一个请求要拉取的数据量不会太大，也不会太小，超到一个限制最大最小的作用 */
  val remoteRequests = splitLocalRemoteBlocks()
  /** Add the remote requests into our queue in a random order */
  fetchRequests ++= Utils.randomize(remoteRequests)
  assert ((0 == reqsInFlight) == (0 == bytesInFlight),
    "expected reqsInFlight = 0 but found reqsInFlight = " + reqsInFlight +
    ", expected bytesInFlight = 0 but found bytesInFlight = " + bytesInFlight)

  /** Send out initial requests for blocks, up to our maxBytesInFlight */
  /** 这个方法是要保证一次发送的请求，不超过 maxBytesInFlight(是 targetRequestSize 的 5 倍) */
  /** 即保证每次发送的拉取数据的请求，拉回来的数据占用的内存不会太大 */
  fetchUpToMaxBytes()

  val numFetches = remoteRequests.size - fetchRequests.size
  logInfo("Started " + numFetches + " remote fetches in" + Utils.getUsedTimeMs(startTime))

  /** Get Local Blocks */
  fetchLocalBlocks()
  logDebug("Got local blocks in " + Utils.getUsedTimeMs(startTime))
}

/** Fetches the next (BlockId, InputStream). If a task fails, the ManagedBuffers */
/** underlying each InputStream will be freed by the cleanup() method registered with the */
/** TaskCompletionListener. However, callers should close() these InputStreams */
/** as soon as they are no longer needed, in order to release memory as early as possible. */
/** */
/** Throws a FetchFailedException if the next block could not be fetched. */
/** 这个方法比较简单，每次返回一个结果，并再调用 fetchUpToMaxBytes 以发送足够的请求 */
override def next(): (BlockId, InputStream) = {
  if (!hasNext) {
    throw new NoSuchElementException
  }

  numBlocksProcessed += 1
  val startFetchWait = System.currentTimeMillis()
  currentResult = results.take()
  val result = currentResult
  val stopFetchWait = System.currentTimeMillis()
  shuffleMetrics.incFetchWaitTime(stopFetchWait - startFetchWait)

  result match {
    case SuccessFetchResult(_, address, size, buf, isNetworkReqDone) =>
      if (address != blockManager.blockManagerId) {
        shuffleMetrics.incRemoteBytesRead(buf.size)
        shuffleMetrics.incRemoteBlocksFetched(1)
      }
      bytesInFlight -= size
      if (isNetworkReqDone) {
        reqsInFlight -= 1
        logDebug("Number of requests in flight " + reqsInFlight)
      }
    case _ =>
  }
  /** Send fetch requests up to maxBytesInFlight */ fetchUpToMaxBytes()

  result match {
    case FailureFetchResult(blockId, address, e) =>
      throwFetchFailedException(blockId, address, e)

    case SuccessFetchResult(blockId, address, _, buf, _) =>
      try {
        (result.blockId, new BufferReleasingInputStream(buf.createInputStream(), this))
      } catch {
        case NonFatal(t) =>
          throwFetchFailedException(blockId, address, t)
      }
  }
}
```

shuffle 过程的读取内容比较简单，主要是 reduce 端的 combine 和 block 的拉取过程的逻辑。所以也写的比较简单。

## 总结

至此，shuffle 的写入和读取的过程基本分析完了。由于用时比较长，且难度比较大，所以存在不少错误之处，后续理解更深入之后再慢慢改正.

# 引用
1. [Spark2.x学习笔记：12、Shuffle机制](https://blog.csdn.net/chengyuqiang/article/details/78171094?locationNum=4&fps=1)
2. [深入理解Spark 2.1 Core （九）：迭代计算和Shuffle的原理与源码分析](https://blog.csdn.net/u011239443/article/details/54981376)
3. [大数据：Spark Shuffle（一）ShuffleWrite:Executor如何将Shuffle的结果进行归并写到数据文件中去](https://blog.csdn.net/raintungli/article/details/70807376)
4. [[Spark性能调优] 第二章：彻底解密Spark的HashShuffle](https://www.cnblogs.com/jcchoiling/p/6431969.html)

