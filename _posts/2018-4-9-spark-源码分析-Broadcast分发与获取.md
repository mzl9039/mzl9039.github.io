---
layout:     post
title:      Broadcast 分发与获取
subtitle:   Broadcast 分布与 Task 获取
date:       2018-04-08 00:00:00
author:     mzl
catalog:    true
tags:
    - spark
    - Broadcast
    - Executor
    - Task
---

{:toc}

# Broadcast 分发与获取

Spark 广播变量 Broadcast 是 spark 分布式计算中很常用的一种变量分发形式， 广播是指将数据从一个节点分发到其它节点，供 Task 计算使用。
它的数据通常是配置文件，map数据集等，但不适合存放过大的数据，这容易导致网络IO性能差或单点压力。

它是 spark 分布式计算优化的一种常见方式：我们知道，分布式计算是以 Task 为单位来完成的。如果某个计算需要一个 100M 大小的常量数据（非 rdd）,
这个计算分 100 个 Task, 运行在 5 个 Executor 上。由于每个 Task 都需要处理这个常量，如果每个 Task 都通过网络去获取这个 100M 的常量，则
关于这个常量的网络IO 为 100 * 100M = 10G；而如果把这个常量广播到各个 Executor 上，每个 Task 去当前的 Executor 上去取这个常量，这个
获取过程是本地的，所以这个常量的网络IO是 5 * 100M = 500M。

由上面的分析，我们知道，广播变量是以节点，或以 Executor 为单位的（TODO：如果一个点节有多个 Executor 呢），而不是以 Task 为单位的。
但 Broadcast 的分发的原理是什么，以及 Task 如果获取这个变量，还不太清楚，这里我们需要梳理一下。

在Spark2 之前的版本中，Broadcast 有两种实现，即 HttpBroadcast 和 TorrentBroadcast。

从网上的博客中，我们大概了解到，HttpBroadcast 方式，是通过一个 HttpServer，所有获取 Broadcast 变量都从这个 server 去获取，这很容易
造成单点压力，因为在 spark2 中已经被删掉了。

而 TorrentBroadcast 方式，则是 p2p 的方式，每个 Executor 从 driver 或最近的 Executor 去获取 Broadcast, 类似下载种子一样，下载的人越多，
下载速度越快，因为每个 Executor 不只是消费者，还是生产者，因此大大减轻了 单点压力。当然，这种形式下，如果 Broadcast 特别大时，依赖有
网络IO性能差的问题存在，但这个问题，就不在本文讨论的范围内了。

## BroadcastManager 初始化

在初始化 SparkContext 时，会调用方式 `createSparkEnv` 方法创建一个 SparkEnv 对象:

    _env = createSparkEnv(_conf, isLocal, listenerBus)

### SparkContext 的 createSparkEnv 方法

在 `createSparkEnv` 方法中，会调用 SparkEnv 的 `createDriverEnv` 方法来创建这个 _env 对象:

```scala
/** This function allows components created by SparkEnv to be mocked in unit tests: */
private[spark] def createSparkEnv(
    conf: SparkConf,
    isLocal: Boolean,
    listenerBus: LiveListenerBus): SparkEnv = {
  SparkEnv.createDriverEnv(conf, isLocal, listenerBus, SparkContext.numDriverCores(master))
}
```
### SparkEnv 的 createDriverEnv 方法

在 SparkEnv 的 `createDriverEnv` 方法中，调用 `create` 方法创建 BroadcastManager 对象：

```scala
/** Create a SparkEnv for the driver. */
private[spark] def createDriverEnv(
    conf: SparkConf,
    isLocal: Boolean,
    listenerBus: LiveListenerBus,
    numCores: Int,
    mockOutputCommitCoordinator: Option[OutputCommitCoordinator] = None): SparkEnv = {
  assert(conf.contains(DRIVER_HOST_ADDRESS),
    s"${DRIVER_HOST_ADDRESS.key} is not set on the driver!")
  assert(conf.contains("spark.driver.port"), "spark.driver.port is not set on the driver!")
  val bindAddress = conf.get(DRIVER_BIND_ADDRESS)
  val advertiseAddress = conf.get(DRIVER_HOST_ADDRESS)
  val port = conf.get("spark.driver.port").toInt
  val ioEncryptionKey = if (conf.get(IO_ENCRYPTION_ENABLED)) {
    Some(CryptoStreamUtils.createKey(conf))
  } else {
    None
  }
  create(
    conf,
    SparkContext.DRIVER_IDENTIFIER,
    bindAddress,
    advertiseAddress,
    port,
    isLocal,
    numCores,
    ioEncryptionKey,
    listenerBus = listenerBus,
    mockOutputCommitCoordinator = mockOutputCommitCoordinator
  )
}

/** Helper method to create a SparkEnv for a driver or an executor. */
/** 这个方法用于在 driver 或 executor 上创建一个 SparkEnv 对象，所以不论是 driver 还是 executor 都有 broadcastManager 对象 */
private def create(
    conf: SparkConf,
    executorId: String,
    bindAddress: String,
    advertiseAddress: String,
    port: Int,
    isLocal: Boolean,
    numUsableCores: Int,
    ioEncryptionKey: Option[Array[Byte]],
    listenerBus: LiveListenerBus = null,
    mockOutputCommitCoordinator: Option[OutputCommitCoordinator] = None): SparkEnv = {

  /** 从这个参数，我们可以理解为 driver 是一个特殊的 executor */
  val isDriver = executorId == SparkContext.DRIVER_IDENTIFIER

  /** Listener bus is only used on the driver */
  if (isDriver) {
    assert(listenerBus != null, "Attempted to create driver SparkEnv with null listener bus!")
  }

  val securityManager = new SecurityManager(conf, ioEncryptionKey)
  ioEncryptionKey.foreach { _ =>
    if (!securityManager.isSaslEncryptionEnabled()) {
      logWarning("I/O encryption enabled without RPC encryption: keys will be visible on the " +
        "wire.")
    }
  }

  val systemName = if (isDriver) driverSystemName else executorSystemName
  val rpcEnv = RpcEnv.create(systemName, bindAddress, advertiseAddress, port, conf,
    securityManager, clientMode = !isDriver)

  /** Figure out which port RpcEnv actually bound to in case the original port is 0 or occupied. */
  /** In the non-driver case, the RPC env's address may be null since it may not be listening */
  /** for incoming connections. */
  if (isDriver) {
    conf.set("spark.driver.port", rpcEnv.address.port.toString)
  } else if (rpcEnv.address != null) {
    conf.set("spark.executor.port", rpcEnv.address.port.toString)
    logInfo(s"Setting spark.executor.port to: ${rpcEnv.address.port.toString}")
  }

  /** Create an instance of the class with the given name, possibly initializing it with our conf */
  def instantiateClass[T](className: String): T = {
    val cls = Utils.classForName(className)
    /** Look for a constructor taking a SparkConf and a boolean isDriver, then one taking just */
    /** SparkConf, then one taking no arguments */
    try {
      cls.getConstructor(classOf[SparkConf], java.lang.Boolean.TYPE)
        .newInstance(conf, new java.lang.Boolean(isDriver))
        .asInstanceOf[T]
    } catch {
      case _: NoSuchMethodException =>
        try {
          cls.getConstructor(classOf[SparkConf]).newInstance(conf).asInstanceOf[T]
        } catch {
          case _: NoSuchMethodException =>
            cls.getConstructor().newInstance().asInstanceOf[T]
        }
    }
  }

  /** Create an instance of the class named by the given SparkConf property, or defaultClassName */
  /** if the property is not set, possibly initializing it with our conf */
  def instantiateClassFromConf[T](propertyName: String, defaultClassName: String): T = {
    instantiateClass[T](conf.get(propertyName, defaultClassName))
  }

  val serializer = instantiateClassFromConf[Serializer](
    "spark.serializer", "org.apache.spark.serializer.JavaSerializer")
  logDebug(s"Using serializer: ${serializer.getClass}")

  val serializerManager = new SerializerManager(serializer, conf, ioEncryptionKey)

  val closureSerializer = new JavaSerializer(conf)

  def registerOrLookupEndpoint(
      name: String, endpointCreator: => RpcEndpoint):
    RpcEndpointRef = {
    if (isDriver) {
      logInfo("Registering " + name)
      rpcEnv.setupEndpoint(name, endpointCreator)
    } else {
      RpcUtils.makeDriverRef(name, conf, rpcEnv)
    }
  }

  /** 这里创建 broadcastManager */
  val broadcastManager = new BroadcastManager(isDriver, conf, securityManager)

  val mapOutputTracker = if (isDriver) {
    new MapOutputTrackerMaster(conf, broadcastManager, isLocal)
  } else {
    new MapOutputTrackerWorker(conf)
  }

  /** Have to assign trackerEndpoint after initialization as MapOutputTrackerEndpoint */
  /** requires the MapOutputTracker itself */
  mapOutputTracker.trackerEndpoint = registerOrLookupEndpoint(MapOutputTracker.ENDPOINT_NAME,
    new MapOutputTrackerMasterEndpoint(
      rpcEnv, mapOutputTracker.asInstanceOf[MapOutputTrackerMaster], conf))

  /** Let the user specify short names for shuffle managers */
  val shortShuffleMgrNames = Map(
    "sort" -> classOf[org.apache.spark.shuffle.sort.SortShuffleManager].getName,
    "tungsten-sort" -> classOf[org.apache.spark.shuffle.sort.SortShuffleManager].getName)
  val shuffleMgrName = conf.get("spark.shuffle.manager", "sort")
  val shuffleMgrClass = shortShuffleMgrNames.getOrElse(shuffleMgrName.toLowerCase, shuffleMgrName)
  val shuffleManager = instantiateClass[ShuffleManager](shuffleMgrClass)

  val useLegacyMemoryManager = conf.getBoolean("spark.memory.useLegacyMode", false)
  val memoryManager: MemoryManager =
    if (useLegacyMemoryManager) {
      new StaticMemoryManager(conf, numUsableCores)
    } else {
      UnifiedMemoryManager(conf, numUsableCores)
    }

  val blockManagerPort = if (isDriver) {
    conf.get(DRIVER_BLOCK_MANAGER_PORT)
  } else {
    conf.get(BLOCK_MANAGER_PORT)
  }

  val blockTransferService =
    new NettyBlockTransferService(conf, securityManager, bindAddress, advertiseAddress,
      blockManagerPort, numUsableCores)

  val blockManagerMaster = new BlockManagerMaster(registerOrLookupEndpoint(
    BlockManagerMaster.DRIVER_ENDPOINT_NAME,
    new BlockManagerMasterEndpoint(rpcEnv, isLocal, conf, listenerBus)),
    conf, isDriver)

  /** NB: blockManager is not valid until initialize() is called later. */
  val blockManager = new BlockManager(executorId, rpcEnv, blockManagerMaster,
    serializerManager, conf, memoryManager, mapOutputTracker, shuffleManager,
    blockTransferService, securityManager, numUsableCores)

  val metricsSystem = if (isDriver) {
    /** Don't start metrics system right now for Driver. */
    /** We need to wait for the task scheduler to give us an app ID. */
    /** Then we can start the metrics system. */
    MetricsSystem.createMetricsSystem("driver", conf, securityManager)
  } else {
    /** We need to set the executor ID before the MetricsSystem is created because sources and */
    /** sinks specified in the metrics configuration file will want to incorporate this executor's */
    /** ID into the metrics they report. */
    conf.set("spark.executor.id", executorId)
    val ms = MetricsSystem.createMetricsSystem("executor", conf, securityManager)
    ms.start()
    ms
  }

  val outputCommitCoordinator = mockOutputCommitCoordinator.getOrElse {
    new OutputCommitCoordinator(conf, isDriver)
  }
  val outputCommitCoordinatorRef = registerOrLookupEndpoint("OutputCommitCoordinator",
    new OutputCommitCoordinatorEndpoint(rpcEnv, outputCommitCoordinator))
  outputCommitCoordinator.coordinatorRef = Some(outputCommitCoordinatorRef)

  val envInstance = new SparkEnv(
    executorId,
    rpcEnv,
    serializer,
    closureSerializer,
    serializerManager,
    mapOutputTracker,
    shuffleManager,
    broadcastManager,
    blockManager,
    securityManager,
    metricsSystem,
    memoryManager,
    outputCommitCoordinator,
    conf)

  /** Add a reference to tmp dir created by driver, we will delete this tmp dir when stop() is */
  /** called, and we only need to do it for driver. Because driver may run as a service, and if we */
  /** don't delete this tmp dir when sc is stopped, then will create too many tmp dirs. */
  if (isDriver) {
    val sparkFilesDir = Utils.createTempDir(Utils.getLocalDir(conf), "userFiles").getAbsolutePath
    envInstance.driverTmpDir = Some(sparkFilesDir)
  }

  envInstance
}
```

### BroadcastManager 初始化

BroadcastManager 实始化时，会调用 `initialize` 方法，并使用 TorrentBroadcastFactory 生成 broadcastFactory 工厂对象，
进行初始化 broadcastFactory 工厂

```scala
private[spark] class BroadcastManager(
    val isDriver: Boolean,
    conf: SparkConf,
    securityManager: SecurityManager)
  extends Logging {

  private var initialized = false
  private var broadcastFactory: BroadcastFactory = null

  /** 初始化时即调用 initialize 方法初始化 BroadcastManager */
  initialize()

  /** Called by SparkContext or Executor before using Broadcast */
  private def initialize() {
    synchronized {
      if (!initialized) {
        /** broadcastFactory 写死为 TorrentBroadcastFactory, 说明 HttpBroadcastFactory 已被删除 */
        broadcastFactory = new TorrentBroadcastFactory
        /** TorrentBroadcastFactory 的 initialize 方法是空方法，没有任何内容 */
        broadcastFactory.initialize(isDriver, conf, securityManager)
        initialized = true
      }
    }
  }

  def stop() {
    broadcastFactory.stop()
  }

  private val nextBroadcastId = new AtomicLong(0)

  def newBroadcast[T: ClassTag](value_ : T, isLocal: Boolean): Broadcast[T] = {
    broadcastFactory.newBroadcast[T](value_, isLocal, nextBroadcastId.getAndIncrement())
  }

  def unbroadcast(id: Long, removeFromDriver: Boolean, blocking: Boolean) {
    broadcastFactory.unbroadcast(id, removeFromDriver, blocking)
  }
}
```

至此，BroadcastManager 的初始化就完成了，剩下的初始化是 Broadcast 变量的初始化。

### Broadcast 广播变量用法

上面提到，BroadcastFactory 是在初始化 env 的过程中完成的，初始化 env 的过程是初始化 sc 或 executor 的一部分。
后面使用广播变量时，只需要调用 sc 的方法 `broadcast` 即可：
    
    /** value 就是需要广播出去的变量 */
    sc.broadcast(value)

### broadcast 方法

使用广播变量时，首先用 SparkContext 的 broadcast 方法包装变量 value，这个方法会调用 broadcastManager 的 newBroadcast
方法创建新的广播变量, broadcastManager 的 newBroadcast 方法又调用 broadcastFactory 对象的 newBroadcast 方法创建新的
广播变量，在 BroadcastFactory 的 newBroadcast 方法中，才真正创建了一个 TorrentBroadcast 广播变量，至此完成广播变量的
创建过程

```scala
/** 这是 SparkContext 类的 broadcast 方法 */
/** Broadcast a read-only variable to the cluster, returning a */
/** [[org.apache.spark.broadcast.Broadcast]] object for reading it in distributed functions. */
/** The variable will be sent to each cluster only once. */
def broadcast[T: ClassTag](value: T): Broadcast[T] = {
  assertNotStopped()
  require(!classOf[RDD[_]].isAssignableFrom(classTag[T].runtimeClass),
    "Can not directly broadcast RDDs; instead, call collect() and broadcast the result.")
  /** 调用 broadcastManager 的 newBroadcast 方法创建新的广播变量 */
  val bc = env.broadcastManager.newBroadcast[T](value, isLocal)
  val callSite = getCallSite
  logInfo("Created broadcast " + bc.id + " from " + callSite.shortForm)
  cleaner.foreach(_.registerBroadcastForCleanup(bc))
  bc
}

/** BroadcastManager 类的 newBroadcast 方法，调用 broadcastFactory 对象的 newBroadcast 方法创建新的广播变量 */
def newBroadcast[T: ClassTag](value_ : T, isLocal: Boolean): Broadcast[T] = {
  broadcastFactory.newBroadcast[T](value_, isLocal, nextBroadcastId.getAndIncrement())
}

/** BroadcastFactory 类的 newBroadcast 方法创建一个新的 TorrentBroadcast 广播变量 */
override def newBroadcast[T: ClassTag](value_ : T, isLocal: Boolean, id: Long): Broadcast[T] = {
  new TorrentBroadcast[T](value_, id)
}
```

## Broadcast 分发

Broadcast 分发，主要是通过 BlockManager 来完成的，对 BlockManager 以及 Block 的分析，在以后的博客中完成，本文不分析，
我们只需要知道 BlockManager 分把广播变量存放在 Block 中，并通过网络分发即可。但我们需要知道广播变量 Broadcast 是如何
转为 Block 的。

广播变量 Broadcast 存入到 Block 中，是在初始化过程中完成的。在上一节的分析中，我们知道创建广播变量，最终是创建了一个
TorrentBroadcast 变量。在这个变量的初始化过程中，即会完成将变量的 value 写入 Block 的流程，即在初始化属性 numBlocks 时，
会调用 writeBlocks 方法完成 value 向 Block 写入的过程:

    /** Total number of blocks this broadcast variable contains. */
    private val numBlocks: Int = writeBlocks(obj)

```scala
/** Divide the object into multiple blocks and put those blocks in the block manager. */
/** */
/** @param value the object to divide */
/** @return number of blocks this broadcast variable is divided into */
private def writeBlocks(value: T): Int = {
  import StorageLevel._
  /** Store a copy of the broadcast variable in the driver so that tasks run on the driver */
  /** do not create a duplicate copy of the broadcast variable's value. */
  /** TODO:在 driver 端保存一个广播变量的备份，这样后面 task 运行在 driver 端时，不需要 */
  /** 再重新创建一个广播变量的备份了, 但这个 putSingle 为什么能做到这个呢？ */
  val blockManager = SparkEnv.get.blockManager
  if (!blockManager.putSingle(broadcastId, value, MEMORY_AND_DISK, tellMaster = false)) {
    throw new SparkException(s"Failed to store $broadcastId in BlockManager")
  }
  /** 把 value 切分为数组，每个数组项后续写入一个 Block */
  val blocks =
    TorrentBroadcast.blockifyObject(value, blockSize, SparkEnv.get.serializer, compressionCodec)
  if (checksumEnabled) {
    checksums = new Array[Int](blocks.length)
  }
  blocks.zipWithIndex.foreach { case (block, i) =>
    if (checksumEnabled) {
      /** 记录每个 Block 的 checksum */
      checksums(i) = calcChecksum(block)
    }
    /** 每个 Block 为一个 Block, 由于分为多个，所以每个 block 理解为一个 piece */
    val pieceId = BroadcastBlockId(id, "piece" + i)
    /** 获取当前数组项的二进制数组 */
    val bytes = new ChunkedByteBuffer(block.duplicate())
    /** 将当前数组项的二进制组件写入 Block */
    if (!blockManager.putBytes(pieceId, bytes, MEMORY_AND_DISK_SER, tellMaster = true)) {
      throw new SparkException(s"Failed to store $pieceId of $broadcastId in local BlockManager")
    }
  }
  blocks.length
}
```

## Task 获取 Broadcast

在代码中，我们会通过 broadcast.value 来获取广播变量的值。由于代码 func 会被序列化并分发到各个节点上，这个序列化及分发
的过程，是在 DagScheduler 的 submitMissingTasks 方法中完成的，先序列化为 taskBinary，并被广播，而且这个 taskBinary 也
被添加到 tasks 里去，并传递到相应的 executor 上去执行。

这个 submitMissingTasks 方法我们截取部分片段放在这里:

```scala
/** TODO: Maybe we can keep the taskBinary in Stage to avoid serializing it multiple times. */
/** Broadcasted binary for the task, used to dispatch tasks to executors. Note that we broadcast */
/** the serialized copy of the RDD and for each task we will deserialize it, which means each */
/** task gets a different copy of the RDD. This provides stronger isolation between tasks that */
/** might modify state of objects referenced in their closures. This is necessary in Hadoop */
/** where the JobConf/Configuration object is not thread-safe. */
/** 这里生成 taskBinary. 注意生成的时候，使用的是 rdd 和 dep/func, 所以序列化的也是这个 */
var taskBinary: Broadcast[Array[Byte]] = null
try {
  /** For ShuffleMapTask, serialize and broadcast (rdd, shuffleDep). */
  /** For ResultTask, serialize and broadcast (rdd, func). */
  val taskBinaryBytes: Array[Byte] = stage match {
    case stage: ShuffleMapStage =>
      JavaUtils.bufferToArray(
        closureSerializer.serialize((stage.rdd, stage.shuffleDep): AnyRef))
    case stage: ResultStage =>
      JavaUtils.bufferToArray(closureSerializer.serialize((stage.rdd, stage.func): AnyRef))
  }

  taskBinary = sc.broadcast(taskBinaryBytes)
} catch {
  /** In the case of a failure during serialization, abort the stage. */
  case e: NotSerializableException =>
    abortStage(stage, "Task not serializable: " + e.toString, Some(e))
    runningStages -= stage

    /** Abort execution */
    return
  case NonFatal(e) =>
    abortStage(stage, s"Task serialization failed: $e\n${Utils.exceptionString(e)}", Some(e))
    runningStages -= stage
    return
}

/** 生成 tasks, 参数之一是 taskBinary. 这个 tasks 后面被提交到 taskScheduler 的 submitTasks 方法 */
val tasks: Seq[Task[_]] = try {
  stage match {
    case stage: ShuffleMapStage =>
      partitionsToCompute.map { id =>
        val locs = taskIdToLocations(id)
        val part = stage.rdd.partitions(id)
        new ShuffleMapTask(stage.id, stage.latestInfo.attemptId,
          taskBinary, part, locs, stage.latestInfo.taskMetrics, properties, Option(jobId),
          Option(sc.applicationId), sc.applicationAttemptId)
      }

    case stage: ResultStage =>
      partitionsToCompute.map { id =>
        val p: Int = stage.partitions(id)
        val part = stage.rdd.partitions(p)
        val locs = taskIdToLocations(id)
        new ResultTask(stage.id, stage.latestInfo.attemptId,
          taskBinary, part, locs, id, properties, stage.latestInfo.taskMetrics,
          Option(jobId), Option(sc.applicationId), sc.applicationAttemptId)
      }
  }
} catch {
  case NonFatal(e) =>
    abortStage(stage, s"Task creation failed: $e\n${Utils.exceptionString(e)}", Some(e))
    runningStages -= stage
    return
}

if (tasks.size > 0) {
  logInfo("Submitting " + tasks.size + " missing tasks from " + stage + " (" + stage.rdd + ")")
  stage.pendingPartitions ++= tasks.map(_.partitionId)
  logDebug("New pending partitions: " + stage.pendingPartitions)
  /** 这里把生成的 tasks 对象提交给 taskScheduler, 后面分发到各个节点去执行 task */
  taskScheduler.submitTasks(new TaskSet(
    tasks.toArray, stage.id, stage.latestInfo.attemptId, jobId, properties))
  stage.latestInfo.submissionTime = Some(clock.getTimeMillis())
} else {
  /** Because we posted SparkListenerStageSubmitted earlier, we should mark */
  /** the stage as completed here in case there are no tasks to run */
  markStageAsFinished(stage, None)

  val debugString = stage match {
    case stage: ShuffleMapStage =>
      s"Stage ${stage} is actually done; " +
        s"(available: ${stage.isAvailable}," +
        s"available outputs: ${stage.numAvailableOutputs}," +
        s"partitions: ${stage.numPartitions})"
    case stage : ResultStage =>
      s"Stage ${stage} is actually done; (partitions: ${stage.numPartitions})"
  }
  logDebug(debugString)

  submitWaitingChildStages(stage)
}
```

### TorrentBroadcast 的 readBroadcastBlock 方法

在各个节点上执行代码时，会将代码逆序列化并调用，在执行代码段中的获取广播变量时，会调用广播变量 broadcast 的 
getValue 方法，而这个方法会调用 TorrentBroadcast 的 readBroadcastBlock 方法，这个方法会从本地或远程获取
当前广播变量的 block,并转换为需要的类型:

```scala
private def readBroadcastBlock(): T = Utils.tryOrIOException {
  TorrentBroadcast.synchronized {
    setConf(SparkEnv.get.conf)
    val blockManager = SparkEnv.get.blockManager
    /** 首先尝试从本地获取，如果本地获取到了，则转换为类型 T 并直接返回 */
    blockManager.getLocalValues(broadcastId) match {
      case Some(blockResult) =>
        if (blockResult.data.hasNext) {
          val x = blockResult.data.next().asInstanceOf[T]
          releaseLock(broadcastId)
          x
        } else {
          throw new SparkException(s"Failed to get locally stored broadcast data: $broadcastId")
        }
      case None =>
        logInfo("Started reading broadcast variable " + id)
        val startTimeMs = System.currentTimeMillis()
        /** 如果本地获取不到，则从 driver 端或其它 executor 端获取 block, 然后转为类型 T 的对象 obj, 并把这个对象存在本地 */
        /** 这样就做到了 p2p, 用的越多，速度越快 */
        val blocks = readBlocks().flatMap(_.getChunks())
        logInfo("Reading broadcast variable " + id + " took" + Utils.getUsedTimeMs(startTimeMs))

        val obj = TorrentBroadcast.unBlockifyObject[T](
          blocks, SparkEnv.get.serializer, compressionCodec)
        // Store the merged copy in BlockManager so other tasks on this executor don't
        // need to re-fetch it.
        val storageLevel = StorageLevel.MEMORY_AND_DISK
        if (!blockManager.putSingle(broadcastId, obj, storageLevel, tellMaster = false)) {
          throw new SparkException(s"Failed to store $broadcastId in BlockManager")
        }
        obj
    }
  }
}
```

### TorrentBroadcast 的 readBlocks 方法

```scala
/** Fetch torrent blocks from the driver and/or other executors. */
private def readBlocks(): Array[ChunkedByteBuffer] = {
  /** Fetch chunks of data. Note that all these chunks are stored in the BlockManager and reported */
  /** to the driver, so other executors can pull these chunks from this executor as well. */
  val blocks = new Array[ChunkedByteBuffer](numBlocks)
  val bm = SparkEnv.get.blockManager

  /** 获取 block 的时候，也是优先多本地去获取，如果本地获取不到，则从远程获取 */
  /** 获取 block 的方法，完全是通过 blockManager 完成的，尤其是通过远程获取 */
  /** blockManager 的 getRemoteBytes 方法，会以类似 p2p 的方式获取 block */
  for (pid <- Random.shuffle(Seq.range(0, numBlocks))) {
    val pieceId = BroadcastBlockId(id, "piece" + pid)
    logDebug(s"Reading piece $pieceId of $broadcastId")
    /** First try getLocalBytes because there is a chance that previous attempts to fetch the */
    /** broadcast blocks have already fetched some of the blocks. In that case, some blocks */
    /** would be available locally (on this executor). */
    bm.getLocalBytes(pieceId) match {
      case Some(block) =>
        blocks(pid) = block
        releaseLock(pieceId)
      case None =>
        bm.getRemoteBytes(pieceId) match {
          case Some(b) =>
            if (checksumEnabled) {
              val sum = calcChecksum(b.chunks(0))
              if (sum != checksums(pid)) {
                throw new SparkException(s"corrupt remote block $pieceId of $broadcastId:" +
                  s" $sum != ${checksums(pid)}")
              }
            }
            /** We found the block from remote executors/driver's BlockManager, so put the block */
            /** in this executor's BlockManager. */
            if (!bm.putBytes(pieceId, b, StorageLevel.MEMORY_AND_DISK_SER, tellMaster = true)) {
              throw new SparkException(
                s"Failed to store $pieceId of $broadcastId in local BlockManager")
            }
            blocks(pid) = b
          case None =>
            throw new SparkException(s"Failed to get $pieceId of $broadcastId")
        }
    }
  }
  blocks
}
```

至此，我们理解了 Task 如何获取广播变量。也梳理了广播变量的分发和获取过程。
