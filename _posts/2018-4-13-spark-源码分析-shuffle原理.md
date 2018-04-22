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
18. DiskBlockObjectWriter: 将 JVM 中的对象直接写入文件的类。这个类允许把数据追加到已经存在的 block。为了效率，它持有底层的文件通道。这个通道会保持 open 状态，
直到 DiskBlockObjectWriter 的 close() 方法被调用。为了防止出现错误(如正在写的过程中出错了), 调用者需要调用方法 revertPartialWritesAndClose 而不是 close 方法，
来自动 revert 那些未提交的 write 操作。

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

spill 方法如下：

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

### Spark shuffle 元数据的消息传递

### Spark shuffle 的读取

# 引用
1. [Spark2.x学习笔记：12、Shuffle机制](https://blog.csdn.net/chengyuqiang/article/details/78171094?locationNum=4&fps=1)
2. [深入理解Spark 2.1 Core （九）：迭代计算和Shuffle的原理与源码分析](https://blog.csdn.net/u011239443/article/details/54981376)
3. [大数据：Spark Shuffle（一）ShuffleWrite:Executor如何将Shuffle的结果进行归并写到数据文件中去](https://blog.csdn.net/raintungli/article/details/70807376)
4. [[Spark性能调优] 第二章：彻底解密Spark的HashShuffle](https://www.cnblogs.com/jcchoiling/p/6431969.html)

