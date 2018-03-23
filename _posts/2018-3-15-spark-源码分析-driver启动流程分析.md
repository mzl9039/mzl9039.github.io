---
layout:     post
title:      "Spark driver 启动流程分析"
subtitle:   "driver 启动"
date:       2018-3-15 16:00:00
author:     mzl
catalog:    true
tags:
    - SparkContext
    - Driver
---

{:toc}
# Spark driver 启动流程分析

我们知道，提交 Spark app 的时候会需要先创建初始化 sc，然后 spark 会启动一个 driver 端，这个 driver 端用来
执行我们日常开发的 app 的 main 方法,并创建 sc（听起来就像是本地代码部分，事实上有细微区别，因为 driver 端
可以不在本地，而在集群上，如 yarn 的 cluster 模式下， driver 端是 yarn 上的一个 app）.

所以我们知道，启动 driver 是在初始化 sc 的时候完成的，这里是我们分析的起点。

## 初始化 DriverEndpoint

从以前的[博客-Spark 任务分发与执行流程](https://mzl9039.github.io/2018/03/05/spark-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-%E4%BB%BB%E5%8A%A1%E5%88%86%E5%8F%91%E4%B8%8E%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B.html)
中提到，TaskSchedulerImpl 和相应的 SchedulerBackend 是在 sc 中初始化完成的。之后调用 taskScheduler 的 start 方法。
并在这个方法里调用了 schedulerBackend 的 start 方法。在集群模式下，无论是 standalone 模式下的 StandaloneSchedulerBackend
还是 yarn 下的 YarnSchedulerBackend, 都继承自类 CoarseGrainedSchedulerBackend，并在 start 方法中初始化 driverEndpoint, 如下：

```scala
override def start() {
  val properties = new ArrayBuffer[(String, String)]
  for ((key, value) <- scheduler.sc.conf.getAll) {
    if (key.startsWith("spark.")) {
      properties += ((key, value))
    }
  }

  /** TODO (prashant) send conf instead of properties */
  driverEndpoint = createDriverEndpointRef(properties)
}

protected def createDriverEndpointRef(
    properties: ArrayBuffer[(String, String)]): RpcEndpointRef = {
  rpcEnv.setupEndpoint(ENDPOINT_NAME, createDriverEndpoint(properties))
}

/** 初始化 DriverEndpoint 类 */
protected def createDriverEndpoint(properties: Seq[(String, String)]): DriverEndpoint = {
  new DriverEndpoint(rpcEnv, properties)
}
```
为了便于举例，我们以 standalone 的集群模式为例说明。此时 schedulerBackend 是 StandaloneSchedulerBackend 类的实例对象。

## 初始化 StandaloneAppClient

在 schedulerBackend 的 start 方法中，会初始化 StandaloneAppClient, 类似的，在 yarn 的 client 模式下，会初始化 Client 类。
即意味着一定要初始化一个 Client 类，但这两个 Client 类并没有公共的父类或接口（除 Logging 外）.接下来看一下类
StandaloneSchedulerBackend 的 start 方法

```scala
override def start() {
  /** 上一节提到，在这里创建了 driverEndpoint  */
  super.start()

  /** SPARK-21159. The scheduler backend should only try to connect to the launcher when in client */
  /** mode. In cluster mode, the code that submits the application to the Master needs to connect */
  /** to the launcher instead. */
  if (sc.deployMode == "client") {
    launcherBackend.connect()
  }

  /** The endpoint for executors to talk to us */
  val driverUrl = RpcEndpointAddress(
    sc.conf.get("spark.driver.host"),
    sc.conf.get("spark.driver.port").toInt,
    CoarseGrainedSchedulerBackend.ENDPOINT_NAME).toString
  val args = Seq(
    "--driver-url", driverUrl,
    "--executor-id", "{{EXECUTOR_ID}}",
    "--hostname", "{{HOSTNAME}}",
    "--cores", "{{CORES}}",
    "--app-id", "{{APP_ID}}",
    "--worker-url", "{{WORKER_URL}}")
  val extraJavaOpts = sc.conf.getOption("spark.executor.extraJavaOptions")
    .map(Utils.splitCommandString).getOrElse(Seq.empty)
  val classPathEntries = sc.conf.getOption("spark.executor.extraClassPath")
    .map(_.split(java.io.File.pathSeparator).toSeq).getOrElse(Nil)
  val libraryPathEntries = sc.conf.getOption("spark.executor.extraLibraryPath")
    .map(_.split(java.io.File.pathSeparator).toSeq).getOrElse(Nil)

  /** When testing, expose the parent class path to the child. This is processed by */
  /** compute-classpath.{cmd,sh} and makes all needed jars available to child processes */
  /** when the assembly is built with the "*-provided" profiles enabled. */
  val testingClassPath =
    if (sys.props.contains("spark.testing")) {
      sys.props("java.class.path").split(java.io.File.pathSeparator).toSeq
    } else {
      Nil
    }

  /** Start executors with a few necessary configs for registering with the scheduler */
  val sparkJavaOpts = Utils.sparkJavaOpts(conf, SparkConf.isExecutorStartupConf)
  val javaOpts = sparkJavaOpts ++ extraJavaOpts
  /** 从类名来看，要创建 CoarseGrainedExecutorBackend 了 */
  val command = Command("org.apache.spark.executor.CoarseGrainedExecutorBackend",
    args, sc.executorEnvs, classPathEntries ++ testingClassPath, libraryPathEntries, javaOpts)
  val appUIAddress = sc.ui.map(_.appUIAddress).getOrElse("")
  val coresPerExecutor = conf.getOption("spark.executor.cores").map(_.toInt)
  /** If we're using dynamic allocation, set our initial executor limit to 0 for now. */
  /** ExecutorAllocationManager will send the real initial limit to the Master later. */
  val initialExecutorLimit =
    if (Utils.isDynamicAllocationEnabled(conf)) {
      Some(0)
    } else {
      None
    }
  /** 初始化 ApplicationDescription, 为后面创建 application 做准备 */
  val appDesc = new ApplicationDescription(sc.appName, maxCores, sc.executorMemory, command,
    appUIAddress, sc.eventLogDir, sc.eventLogCodec, coresPerExecutor, initialExecutorLimit)
  /** 初始化 StandaloneAppClient */
  client = new StandaloneAppClient(sc.env.rpcEnv, masters, appDesc, this, conf)
  /** 启动 StandaloneAppClient，在启动方法里，会初始化 ClientEndpoint */
  client.start()
  launcherBackend.setState(SparkAppHandle.State.SUBMITTED)
  /** 等待注册完成, 这里的注册可能包括 driver 信息的注册，app 的注册和 executor 的注册 */
  waitForRegistration()
  launcherBackend.setState(SparkAppHandle.State.RUNNING)
}
```

在前面的[博客](https://mzl9039.github.io/2018/03/13/spark-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-Master-Worker%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B.html)中，我们简单介绍了如何触发 onStart 方法，这里也是类似的，我们不再赘述了，我们清楚，后面
我们关注 ClientEndpoint 注册到 master 节点.

## ClientEndpoint 注册到 Master 节点

在 ClientEndpoint 的 onStart 方法被调用后，会调用方法 registerWithMaster 向 Master 节点注册 app，如下：

```scala
/** 触发 onStart 方法后，向 master 注册 application 信息 */
override def onStart(): Unit = {
  try {
    registerWithMaster(1)
  } catch {
    case e: Exception =>
      logWarning("Failed to connect to master", e)
      markDisconnected()
      stop()
  }
}

/** Register with all masters asynchronously. It will call `registerWithMaster` every */
/** REGISTRATION_TIMEOUT_SECONDS seconds until exceeding REGISTRATION_RETRIES times. */
/** Once we connect to a master successfully, all scheduling work and Futures will be cancelled. */
/** */
/** nthRetry means this is the nth attempt to register with master. */
private def registerWithMaster(nthRetry: Int) {
  /** 首先尝试向所有节点注册 application */
  registerMasterFutures.set(tryRegisterAllMasters())
  /** 这里是注册失败后的重试机制，多次尝试向 master 节点注册 app, 这里我们忽略掉 */
  registrationRetryTimer.set(registrationRetryThread.schedule(new Runnable {
    override def run(): Unit = {
      if (registered.get) {
        registerMasterFutures.get.foreach(_.cancel(true))
        registerMasterThreadPool.shutdownNow()
      } else if (nthRetry >= REGISTRATION_RETRIES) {
        markDead("All masters are unresponsive! Giving up.")
      } else {
        registerMasterFutures.get.foreach(_.cancel(true))
        registerWithMaster(nthRetry + 1)
      }
    }
  }, REGISTRATION_TIMEOUT_SECONDS, TimeUnit.SECONDS))
}

/**  Register with all masters asynchronously and returns an array `Future`s for cancellation. */
private def tryRegisterAllMasters(): Array[JFuture[_]] = {
  /** 启动 master url 里所有的 master 节点, 如果当前 master 节点为 standby，则跳过不处理 application 的注册请求 */
  for (masterAddress <- masterRpcAddresses) yield {
    registerMasterThreadPool.submit(new Runnable {
      override def run(): Unit = try {
        if (registered.get) {
          return
        }
        logInfo("Connecting to master " + masterAddress.toSparkURL + "...")
        val masterRef = rpcEnv.setupEndpointRef(masterAddress, Master.ENDPOINT_NAME)
        /** 向 master 节点提交请求，触发其 RegisterApplication 事件 */
        masterRef.send(RegisterApplication(appDescription, self))
      } catch {
        case ie: InterruptedException => // Cancelled
        case NonFatal(e) => logWarning(s"Failed to connect to master $masterAddress", e)
      }
    })
  }
}
```

## Master 的 RegisterApplication 事件

ClientEndpoint 向 Master 发送 RegisterApplication 事件请求，准备注册 Application

```scala
case RegisterApplication(description, driver) =>
  /** TODO Prevent repeated registrations from some driver */
  /** 若当前 Master 节点为 standby，则忽略注册 app 的事件 */
  if (state == RecoveryState.STANDBY) {
    /** ignore, don't send response */
  } else {
    logInfo("Registering app " + description.name)
    /** 创建 app */
    val app = createApplication(description, driver)
    /** 注册 app */
    registerApplication(app)
    logInfo("Registered app " + description.name + " with ID " + app.id)
    /** 将 app 信息持久化 */
    persistenceEngine.addApplication(app)
    /** 向 driver 端发送 RegisteredApplication 事件 */
    driver.send(RegisteredApplication(app.id, self))
    /** 启动 executor */
    schedule()
  }
```

在这里，我们顺便关注一下 ClientEndpoint 的 RegisteredApplication 事件
这个事件只是 client 端对 app 已经注册后的状态的更新，并没有很重要的方式

```scala
case RegisteredApplication(appId_, masterRef) =>
  /** FIXME How to handle the following cases? */
  /** 1. A master receives multiple registrations and sends back multiple */
  /** RegisteredApplications due to an unstable network. */
  /** 2. Receive multiple RegisteredApplication from different masters because the master is */
  /** changing. */
  appId.set(appId_)
  registered.set(true)
  master = Some(masterRef)
  listener.connected(appId.get)
```

## Master 的 schedule 方法

这个方法里做了几个重要工作，如 launchDriver 和 startExecutorsOnWorkers, 其中:
1. launchDriver 方法是在所有有效的 Worker 节点启动一个相应的 driver 线程，处理该 driver 有关的信息。
2. startExecutorsOnWorkers 方法在 worker 节点启动 executor.

```scala
/** Schedule the currently available resources among waiting apps. This method will be called */
/** every time a new app joins or resource availability changes. */
private def schedule(): Unit = {
  if (state != RecoveryState.ALIVE) {
    return
  }
  /** Drivers take strict precedence over executors */
  val shuffledAliveWorkers = Random.shuffle(workers.toSeq.filter(_.state == WorkerState.ALIVE))
  val numWorkersAlive = shuffledAliveWorkers.size
  var curPos = 0
  for (driver <- waitingDrivers.toList) { // iterate over a copy of waitingDrivers
    /** We assign workers to each waiting driver in a round-robin fashion. For each driver, we */
    /** start from the last worker that was assigned a driver, and continue onwards until we have */
    /** explored all alive workers. */
    var launched = false
    var numWorkersVisited = 0
    while (numWorkersVisited < numWorkersAlive && !launched) {
      val worker = shuffledAliveWorkers(curPos)
      numWorkersVisited += 1
      if (worker.memoryFree >= driver.desc.mem && worker.coresFree >= driver.desc.cores) {
        /** 如果 worker 节点的资源满足 driver 线程需要的资源，则在 Worker 节点启动一个 driver 线程 */
        launchDriver(worker, driver)
        waitingDrivers -= driver
        launched = true
      }
      curPos = (curPos + 1) % numWorkersAlive
    }
  }
  /** 在 worker 节点启动 executor, 为接下来的计算做准备 */
  startExecutorsOnWorkers()
}
```

## Master 的 launchDriver 方法

launchDriver 方法主要是触发了 worker 节点的 LaunchDriver 事件

```scala
private def launchDriver(worker: WorkerInfo, driver: DriverInfo) {
  logInfo("Launching driver " + driver.id + " on worker " + worker.id)
  worker.addDriver(driver)
  driver.worker = Some(worker)
  /** 触发 worker 节点的 LaunchDriver 事件 */
  worker.endpoint.send(LaunchDriver(driver.id, driver.desc))
  driver.state = DriverState.RUNNING
}

/** Worker 节点的 LaunchDriver 事件 */
/** 可以知道该事件是在 Worker 节点启动了一个线程, 该线程保留了 driver 节点的信息 */
case LaunchDriver(driverId, driverDesc) =>
  logInfo(s"Asked to launch driver $driverId")
  val driver = new DriverRunner(
    conf,
    driverId,
    workDir,
    sparkHome,
    driverDesc.copy(command = Worker.maybeUpdateSSLSettings(driverDesc.command, conf)),
    self,
    workerUri,
    securityMgr)
  drivers(driverId) = driver
  driver.start()

  coresUsed += driverDesc.cores
  memoryUsed += driverDesc.mem
```
