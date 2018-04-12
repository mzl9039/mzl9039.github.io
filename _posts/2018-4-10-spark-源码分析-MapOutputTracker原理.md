---
layout:     post
title:      MapOutputTracker 原理
subtitle:   
date:       2018-04-09 00:00:00
author:     mzl
catalog:    true
tags:
    - spark
    - MapOutputTracker
    - MapOutputTrackerMaster
    - MapOutputTrackerWorker
---

{:toc}

# MapOutputTracker 说明

spark 中每个 stage 的每个 map/reduce 任务都会有唯一的标识，分别为 mapId 和 reduceId.

spark 中每个 shuffle 过程都有唯一的标识，称为 shuffleId.

MapOutputTracker 用于跟踪 stage 的 map 阶段的任务输出的位置，这个位置便于 reduce 阶段任务获取
中址以及中间输出结果。由于每个 reduce 任务的输入可能是多个 map 任务的输出，reduce 会到各个 map
任务所在节点去拉 Block,即 shuffle.

由于 driver 端和 executor 端的作用不同，因而实现方式也不同，分别为 MapOutputTrackerMaster 和
MapOutputTrackerWorker.

由于 MapOutputTracker 是用来记录 shuffle 过程中的任何的输出信息的，所以我们尽量通过任务提交和运行
的流程来关注 MapOutputTracker 的调用情况。

## MapOutputTracker 的初始化

在[博客-Broadcast 分发与获取](https://mzl9039.github.io/2018/04/08/spark-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-Broadcast%E5%88%86%E5%8F%91%E4%B8%8E%E8%8E%B7%E5%8F%96.html)中我们提到，
无论是 sc 初始化(可以理解为 driver 端初始化，会调用方法 createDriverEnv)，
还是 CoarseGrainedExecutorBackend 初始化(可以理解为 executor 端初始化，会调用方法 createExecutorEnv),
最终都会去初始化一个 SparkEnv 对象

```scala
/** 根据是否为 driver 端生成相应的 MapOutputTracker 类，注意两者的区别 */
val mapOutputTracker = if (isDriver) {
  new MapOutputTrackerMaster(conf, broadcastManager, isLocal)
} else {
  new MapOutputTrackerWorker(conf)
}

/** Have to assign trackerEndpoint after initialization as MapOutputTrackerEndpoint */
/** requires the MapOutputTracker itself */
/** 这句话说明即使是 executor 端的 mapOutputTracker，其 trackerEndpoint 也是指向 MapOutputTrackerMasterEndpoint 的 */
/** 当然，driver 端的 mapOutputTracker 也是如此 */
mapOutputTracker.trackerEndpoint = registerOrLookupEndpoint(MapOutputTracker.ENDPOINT_NAME,
  new MapOutputTrackerMasterEndpoint(
    rpcEnv, mapOutputTracker.asInstanceOf[MapOutputTrackerMaster], conf))
```

## Driver 端 MapOutputTracker 的调用

MapOutputTrackerMaster 在 Driver 端完成初始化后，其主要的调用是在 stage 划分开始的，后面的 task 提交运行和 task 结果返回
Driver 可能也会涉及的。我们从 stage 划分阶段开始分析。

从[博客-Spark stage 划分](https://mzl9039.github.io/2018/04/07/spark-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-stage%E5%88%92%E5%88%86.html)中我们知道，
初始化 sc 完成后，dagScheduler 已经初始化完成，而 mapOutputTracker 是 dagScheduler 的一个成员属性。

### 注册 Shuffle

而提交 job 到 dagScheduler 后，最终是由 dagScheduler 的方法 handleJobSubmitted 来划分 stage 并提交 stage. 在划分 stage 过程中，那些 ShuffleMapStage
会被注册到 mapOutputTracker 中

```scala
/** Creates a ShuffleMapStage that generates the given shuffle dependency's partitions. If a */
/** previously run stage generated the same shuffle data, this function will copy the output */
/** locations that are still available from the previous shuffle to avoid unnecessarily */
/** regenerating data. */
def createShuffleMapStage(shuffleDep: ShuffleDependency[_, _, _], jobId: Int): ShuffleMapStage = {
  val rdd = shuffleDep.rdd
  val numTasks = rdd.partitions.length
  val parents = getOrCreateParentStages(rdd, jobId)
  val id = nextStageId.getAndIncrement()
  val stage = new ShuffleMapStage(id, rdd, numTasks, parents, jobId, rdd.creationSite, shuffleDep)

  stageIdToStage(id) = stage
  shuffleIdToMapStage(shuffleDep.shuffleId) = stage
  updateJobIdStageIdMaps(jobId, stage)

  /** 这里尝试去注册 shuffle, 要先判断 shuffle 是否已经注册，因为前面可能有已经开始运行的 stage 注册了这个 shuffle */
  if (mapOutputTracker.containsShuffle(shuffleDep.shuffleId)) {
    /** A previously run stage generated partitions for this shuffle, so for each output */
    /** that's still available, copy information about that output location to the new stage */
    /** (so we don't unnecessarily re-compute that data). */
    /** 一个前面已经开始运行的 stage 已经生成了这个 shuffle 的 partitions，所以对每个仍然可用的 output, */
    /** 只需要把相关信息（如 output location）拷贝到新的 stage，而不需要再次计算这些数据 */
    val serLocs = mapOutputTracker.getSerializedMapOutputStatuses(shuffleDep.shuffleId)
    val locs = MapOutputTracker.deserializeMapStatuses(serLocs)
    (0 until locs.length).foreach { i =>
      if (locs(i) ne null) {
        /** locs(i) will be null if missing */
        stage.addOutputLoc(i, locs(i))
      }
    }
  } else {
    /** Kind of ugly: need to register RDDs with the cache and map output tracker here */
    /** since we can't do it in the RDD constructor because # of partitions is unknown */
    /** 源码作者说这里的实现有些丑，因为我们无法从 RDD 的构造函数里获取 partitions 的信息，
    /** 所以需要注册 rdd 和 map output tracker，不太明白 */
    logInfo("Registering RDD " + rdd.id + " (" + rdd.getCreationSite + ")")
    /** 注册 shuffle */
    mapOutputTracker.registerShuffle(shuffleDep.shuffleId, rdd.partitions.length)
  }
  stage
}

/** MapOutputTrackerMaster 类注册 shuffle 的方法 */
def registerShuffle(shuffleId: Int, numMaps: Int) {
  /** 这个方法把 shuffleId 放到了 mapStatuses 里，同时生成了一个锁，放在了 shuffleIdLocks 里 */
  if (mapStatuses.put(shuffleId, new Array[MapStatus](numMaps)).isDefined) {
    throw new IllegalArgumentException("Shuffle ID " + shuffleId + " registered twice")
  }
  /** add in advance */
  shuffleIdLocks.putIfAbsent(shuffleId, new Object())
}
```

### 获取 Map 输出结果

当某个 task 执行结束后：
1. driver 端的 taskScheduler 会收到 executor 发送的远程事件 statusUpdate，其中 TaskState 被标识为 FINISHED，
2. 然后 taskScheduler 调用 taskResultGetter 的方法 enqueueSuccessfulTask，
3. enqueueSuccessfulTask 调用 sheduler(SchedulerBackend) 的方法 handleSuccessfulTask，
4. handleSuccessfulTask 调用 taskManager 的方法 handleSuccessfulTask，
5. handleSuccessfulTask 调用 dagScheduler 的方法 taskEnded，
6. taskEnded 触发 dagScheduler 的事件 CompletionEvent
7. CompletionEvent 事件调用 dagScheduler 的方法 handleTaskCompletion
8. handleTaskCompletion 方法里将 map 结果注册到 mapOutputTracker 等。

这里取 handleTaskCompletion 方法里的代码片段，这里涉及到的关于 spark Shuffle 的过程和实现原理，后面的文章再分析.

```scala
case smt: ShuffleMapTask =>
  /** 当前的 task 对应的 stage 是一个 ShuffleMapStage */
  val shuffleStage = stage.asInstanceOf[ShuffleMapStage]
  updateAccumulators(event)
  /** event 是事件参数，保存了 task 的元数据信息，如 taskId, result 等等 */
  val status = event.result.asInstanceOf[MapStatus]
  /** MapStatus 是 ShuffleMapTask 返回给 scheduler 的结果。它包含了 task 运行所在的 block manager 的地址， */
  /** 以及每个 reducer 的输出的 size，用于传递到 reduce task, TODO: 用于传递到 recude task 是什么意思? */
  val execId = status.location.executorId
  logDebug("ShuffleMapTask finished on " + execId)
  if (failedEpoch.contains(execId) && smt.epoch <= failedEpoch(execId)) {
    logInfo(s"Ignoring possibly bogus $smt completion from executor $execId")
  } else {
    shuffleStage.addOutputLoc(smt.partitionId, status)
  }

  if (runningStages.contains(shuffleStage) && shuffleStage.pendingPartitions.isEmpty) {
    markStageAsFinished(shuffleStage)
    logInfo("looking for newly runnable stages")
    logInfo("running: " + runningStages)
    logInfo("waiting: " + waitingStages)
    logInfo("failed: " + failedStages)

    /** We supply true to increment the epoch number here in case this is a */
    /** recomputation of the map outputs. In that case, some nodes may have cached */
    /** locations with holes (from when we detected the error) and will need the */
    /** epoch incremented to refetch them. */
    /** TODO: Only increment the epoch number if this is not the first time */
    /**       we registered these map outputs. */
    /** 将当前 ShuffleMapStage 中每个分区的计算结果（并非真实的数据，而是这些数据所在的位置/大小等元数据信息） */
    /** 进行保存，并增加 epoch 编号。这样依赖该 ShuffleMapStage 的其它 ShuffleMapStage/ResultStage 就可以通过这 */
    /** 些元数据信息获取其需要的数据 */
    mapOutputTracker.registerMapOutputs(
      shuffleStage.shuffleDep.shuffleId,
      shuffleStage.outputLocInMapOutputTrackerFormat(),
      changeEpoch = true)

    clearCacheLocs()

    if (!shuffleStage.isAvailable) {
      /** Some tasks had failed; let's resubmit this shuffleStage */
      /** TODO: Lower-level scheduler should also deal with this */
      logInfo("Resubmitting " + shuffleStage + " (" + shuffleStage.name +
        ") because some of its tasks had failed: " +
        shuffleStage.findMissingPartitions().mkString(", "))
      submitStage(shuffleStage)
    } else {
      /** Mark any map-stage jobs waiting on this stage as finished */
      if (shuffleStage.mapStageJobs.nonEmpty) {
        /** TODO：这里的 mapStageJobs 是什么含义和作用? */
        val stats = mapOutputTracker.getStatistics(shuffleStage.shuffleDep)
        for (job <- shuffleStage.mapStageJobs) {
          markMapStageJobAsFinished(job, stats)
        }
      }
      submitWaitingChildStages(shuffleStage)
    }
  }
```

## Executor 端 MapOutputTracker 的调用

在 Executor 上执行 task 时，需要读取前面的 ShuffleMapStage 的数据，由于只有 ResultStage 才会触发计算，所以执行 ResultTask 时才真正去拉取数据
进行计算，前面的 ShuffleMapStage 都只是把想着数据的元数据信息存储起来。这里需要注意的是，shuffle 过程中，读取数据是通过获取 ShuffleReader 的
read 方法来读取的, 同样，spark shuffle 的原理和逻辑，以后再分析。

```scala
/** Read the combined key-values for this reduce task */
override def read(): Iterator[Product2[K, C]] = {
  /** 这里的 mapOutputTracker 调用了 getMapSizesByExecutorId 方法, 这个方法返回了一组二元组序列 Seq[(BlockManagerId, Seq[(BlockId, Long)])], */
  /** 第一项为 BlockManagerId, 第二项为存储于该 BlockManager 上的一组 shuffle blocks, 这里的 getMapSizesByExecutorId 会获取 ShuffleMapStage */
  /** 的 输出信息 MapStatus */
  val wrappedStreams = new ShuffleBlockFetcherIterator(
    context,
    blockManager.shuffleClient,
    blockManager,
    mapOutputTracker.getMapSizesByExecutorId(handle.shuffleId, startPartition, endPartition),
    serializerManager.wrapStream,
    /** Note: we use getSizeAsMb when no suffix is provided for backwards compatibility */
    SparkEnv.get.conf.getSizeAsMb("spark.reducer.maxSizeInFlight", "48m") * 1024 * 1024,
    SparkEnv.get.conf.getInt("spark.reducer.maxReqsInFlight", Int.MaxValue),
    SparkEnv.get.conf.get(config.REDUCER_MAX_REQ_SIZE_SHUFFLE_TO_MEM),
    SparkEnv.get.conf.getBoolean("spark.shuffle.detectCorrupt", true))

  val serializerInstance = dep.serializer.newInstance()

  /** Create a key/value iterator for each stream */
  val recordIter = wrappedStreams.flatMap { case (blockId, wrappedStream) =>
    /** Note: the asKeyValueIterator below wraps a key/value iterator inside of a */
    /** NextIterator. The NextIterator makes sure that close() is called on the */
    /** underlying InputStream when all records have been read. */
    serializerInstance.deserializeStream(wrappedStream).asKeyValueIterator
  }

  /** Update the context task metrics for each record read. */
  val readMetrics = context.taskMetrics.createTempShuffleReadMetrics()
  val metricIter = CompletionIterator[(Any, Any), Iterator[(Any, Any)]](
    recordIter.map { record =>
      readMetrics.incRecordsRead(1)
      record
    },
    context.taskMetrics().mergeShuffleReadMetrics())

  /** An interruptible iterator must be used here in order to support task cancellation */
  val interruptibleIter = new InterruptibleIterator[(Any, Any)](context, metricIter)

  val aggregatedIter: Iterator[Product2[K, C]] = if (dep.aggregator.isDefined) {
    if (dep.mapSideCombine) {
      /** We are reading values that are already combined */
      val combinedKeyValuesIterator = interruptibleIter.asInstanceOf[Iterator[(K, C)]]
      dep.aggregator.get.combineCombinersByKey(combinedKeyValuesIterator, context)
    } else {
      /** We don't know the value type, but also don't care -- the dependency *should* */
      /** have made sure its compatible w/ this aggregator, which will convert the value */
      /** type to the combined type C */
      val keyValuesIterator = interruptibleIter.asInstanceOf[Iterator[(K, Nothing)]]
      dep.aggregator.get.combineValuesByKey(keyValuesIterator, context)
    }
  } else {
    require(!dep.mapSideCombine, "Map-side combine without Aggregator specified!")
    interruptibleIter.asInstanceOf[Iterator[Product2[K, C]]]
  }

  /** Sort the output if there is a sort ordering defined. */
  dep.keyOrdering match {
    case Some(keyOrd: Ordering[K]) =>
      /** Create an ExternalSorter to sort the data. */
      val sorter =
        new ExternalSorter[K, C, C](context, ordering = Some(keyOrd), serializer = dep.serializer)
      sorter.insertAll(aggregatedIter)
      context.taskMetrics().incMemoryBytesSpilled(sorter.memoryBytesSpilled)
      context.taskMetrics().incDiskBytesSpilled(sorter.diskBytesSpilled)
      context.taskMetrics().incPeakExecutionMemory(sorter.peakMemoryUsedBytes)
      CompletionIterator[Product2[K, C], Iterator[Product2[K, C]]](sorter.iterator, sorter.stop())
    case None =>
      aggregatedIter
  }
}
```

### 获取 MapStatus

在集群中，getMapSizesByExecutorId 这个方法是在 Executor 上调用的，当前的 mapOutputTracker 是一个 MapOutputTrackerWorker 对象，
因此 getMapSizesByExecutorId 的实现如下：

```scala
override def getMapSizesByExecutorId(shuffleId: Int, startPartition: Int, endPartition: Int)
    : Seq[(BlockManagerId, Seq[(BlockId, Long)])] = {
  logDebug(s"Fetching outputs for shuffle $shuffleId, partitions $startPartition-$endPartition")
  /** 这里的 getStatuses 方法就是获取 MapStatus 的地方 */
  val statuses = getStatuses(shuffleId)
  try {
    MapOutputTracker.convertMapStatuses(shuffleId, startPartition, endPartition, statuses)
  } catch {
    case e: MetadataFetchFailedException =>
      /** We experienced a fetch failure so our mapStatuses cache is outdated; clear it: */
      mapStatuses.clear()
      throw e
  }
}
```

下面分析 MapOutputTrackerWorker 的 getStatuses 方法：
1. 尝试从本地缓存 mapStatuses 中获取 mapStatus, 若存在则直接返回，否则从远程拉取;
2. fetching 存储了当前正在 fetch 的 shuffleId，若当前的 shuffleId 在 fetching 集合中，则阻塞线程等待；否则将当前 shuffleId 添加到 fetching；
3. 调用 askTracker 方法，触发 MapOutputTrackerMaster 的事件 GetMapOutputStatuses，阻塞线程等待结果；
4. MapOutputTrackerMaster 调用方法 getSerializedMapOutputStatuses,查询本地缓存中 shuffle 对应的 map output status 信息.
    1. 
5. 本地的 mapOutputTracker（实际上是 MapOutputTrackerWorker）的 askTracker 接收到数据后，将数据反序列化，并添加到本地缓存 mapStatuses.
6. 根据执行的分区范围 [startPartition, endPartition] 将返回的结果 Array[MapStatus] map 为 Seq[(BlockManagerId, Seq[(BlockId, Long)])]

```scala
/** Get or fetch the array of MapStatuses for a given shuffle ID. NOTE: clients MUST synchronize */
/** on this array when reading it, because on the driver, we may be changing it in place. */
/** */
/** (It would be nice to remove this restriction in the future.) */
private def getStatuses(shuffleId: Int): Array[MapStatus] = {
  val statuses = mapStatuses.get(shuffleId).orNull
  if (statuses == null) {
    logInfo("Don't have map outputs for shuffle " + shuffleId + ", fetching them")
    val startTime = System.currentTimeMillis
    var fetchedStatuses: Array[MapStatus] = null
    fetching.synchronized {
      /** Someone else is fetching it; wait for them to be done */
      while (fetching.contains(shuffleId)) {
        try {
          fetching.wait()
        } catch {
          case e: InterruptedException =>
        }
      }

      /** Either while we waited the fetch happened successfully, or */
      /** someone fetched it in between the get and the fetching.synchronized. */
      fetchedStatuses = mapStatuses.get(shuffleId).orNull
      if (fetchedStatuses == null) {
        /** We have to do the fetch, get others to wait for us. */
        fetching += shuffleId
      }
    }

    if (fetchedStatuses == null) {
      /** We won the race to fetch the statuses; do so */
      logInfo("Doing the fetch; tracker endpoint = " + trackerEndpoint)
      /** This try-finally prevents hangs due to timeouts: */
      try {
        val fetchedBytes = askTracker[Array[Byte]](GetMapOutputStatuses(shuffleId))
        fetchedStatuses = MapOutputTracker.deserializeMapStatuses(fetchedBytes)
        logInfo("Got the output locations")
        mapStatuses.put(shuffleId, fetchedStatuses)
      } finally {
        fetching.synchronized {
          fetching -= shuffleId
          fetching.notifyAll()
        }
      }
    }
    logDebug(s"Fetching map output statuses for shuffle $shuffleId took " +
      s"${System.currentTimeMillis - startTime} ms")

    if (fetchedStatuses != null) {
      fetchedStatuses
    } else {
      logError("Missing all output locations for shuffle " + shuffleId)
      throw new MetadataFetchFailedException(
        shuffleId, -1, "Missing all output locations for shuffle " + shuffleId)
    }
  } else {
    statuses
  }
}
```

## MapOutputTracker 相关类源码分析

# 引用：
1. [spark学习-35-Spark的Map任务输出跟踪器MapOutputTracker](https://blog.csdn.net/qq_21383435/article/details/78603123)
2. [Spark MapOutputTracker原理](https://blog.csdn.net/luofenghan/article/details/78591305)
