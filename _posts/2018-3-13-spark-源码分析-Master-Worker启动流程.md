---
layout:     post
title:      "Spark Master-Worker启动流程"
subtitle:   ""
date:       2018-3-13 16:00:00
author:     mzl
catalog:    true
tags:
    - Master
    - Worker 
---

{:toc}
# Spark Master/Worker启动流程

在分析提交 task 的流程中，被 executor 的启动流程搞混了，打算写几篇关于启动的文章，自然考虑从 Master 和 Worker 的启动写起。
在启动过程中，主要是消息的传递和通信，本身其实不复杂。

## 从启动脚本说起

对 spark 启动有一些了解的人知道，spark 启动是从脚本文件 start-all.sh 开始的，如下(只粘贴了关键的语句)：

```shell
# 前面的准备工作语句跳过
......
# 下面这两句才是我们要关心的脚本
# 启动 spark master
# Start Master
"${SPARK_HOME}/sbin"/start-master.sh

# 启动 spark slaves
# Start Workers
"${SPARK_HOME}/sbin"/start-slaves.sh
```

启动 spark master 的脚本是 start-master.sh，如下：

```shell
# 前面的准备工作语句跳过
......

# NOTE: This exact class name is matched downstream by SparkSubmit.
# 注意：这个类是启动 spark master 的类！！！后面会讲到，启动 spark master，是通过这个类的 main 方法启动的
# Any changes need to be reflected there.
CLASS="org.apache.spark.deploy.master.Master"

# 跳过部分代码
......

ORIGINAL_ARGS="$@"

if [ "$SPARK_MASTER_PORT" = "" ]; then
  SPARK_MASTER_PORT=7077
fi

# 跳过部分代码
......

if [ "$SPARK_MASTER_WEBUI_PORT" = "" ]; then
  SPARK_MASTER_WEBUI_PORT=8080
fi

# 通过进程启动类 $CLASS, 即 org.apache.spark.deploy.master.Master
"${SPARK_HOME}/sbin"/spark-daemon.sh start $CLASS 1 \
  --host $SPARK_MASTER_HOST --port $SPARK_MASTER_PORT --webui-port $SPARK_MASTER_WEBUI_PORT \
  $ORIGINAL_ARGS
```

启动 spark worker 的脚本是 start-slaves.sh，如下:

```shell
# 前面的准备工作语句跳过
......

# Find the port number for the master
if [ "$SPARK_MASTER_PORT" = "" ]; then
  SPARK_MASTER_PORT=7077
fi

# 跳过部分语句
......

# 我们看到，这里启动 spark worker 关键是 slaves.sh 和 start-slave.sh，注意后面跟了 spark:// 这样的 spark url！
# 对于启动流程而言， slaves.sh 重要性较低，跳过，但注意 slaves.sh 需要知道在哪些节点起 worker! 重点关注 start-slave.sh 文件
# Launch the slaves
"${SPARK_HOME}/sbin/slaves.sh" cd "${SPARK_HOME}" \; "${SPARK_HOME}/sbin/start-slave.sh" "spark://$SPARK_MASTER_HOST:$SPARK_MASTER_PORT"
```

启动 spark worker 的脚本 start-slave.sh，如下：

```shell
if [ -z "${SPARK_HOME}" ]; then
  export SPARK_HOME="$(cd "`dirname "$0"`"/..; pwd)"
fi

# NOTE: This exact class name is matched downstream by SparkSubmit.
# Any changes need to be reflected there.
# 指定了要启动的类
CLASS="org.apache.spark.deploy.worker.Worker"

# 跳过部分语句
......

# First argument should be the master; we need to store it aside because we may
# need to insert arguments between it and the other arguments
# 第一个参数应该指定 master
MASTER=$1
shift

# Determine desired worker port
if [ "$SPARK_WORKER_WEBUI_PORT" = "" ]; then
  SPARK_WORKER_WEBUI_PORT=8081
fi

# Start up the appropriate number of workers on this machine.
# quick local function to start a worker
# 每个节点上可以起不只一个 slave 实例，因此这里定义了一个函数，若在一个节点起多个实例，
# 则循环调用这个函数
function start_instance {
  WORKER_NUM=$1
  shift

  if [ "$SPARK_WORKER_PORT" = "" ]; then
    PORT_FLAG=
    PORT_NUM=
  else
    PORT_FLAG="--port"
    PORT_NUM=$(( $SPARK_WORKER_PORT + $WORKER_NUM - 1 ))
  fi
  WEBUI_PORT=$(( $SPARK_WORKER_WEBUI_PORT + $WORKER_NUM - 1 ))

  # 我们看到，起 Worker 实例的时候，依然是用 start-daemon.sh 启动，启动的类是 Worker 类
  "${SPARK_HOME}/sbin"/spark-daemon.sh start $CLASS $WORKER_NUM \
     --webui-port "$WEBUI_PORT" $PORT_FLAG $PORT_NUM $MASTER "$@"
}

# 若没指定 SPARK_WORKER_INSTANCES 变量，则每个节点启动一个实例，否则指指定的个数启动。
if [ "$SPARK_WORKER_INSTANCES" = "" ]; then
  start_instance 1 "$@"
else
  for ((i=0; i<$SPARK_WORKER_INSTANCES; i++)); do
    start_instance $(( 1 + $i )) "$@"
  done
fi
```

我们不再分析 start-daemon.sh 脚本了因为我们已经基本清楚了, 通过脚本启动 spark master/worker 的过程，现在可以考虑启动 master 的流程了。

## 写在启动前面的话

由于 Master 和 Worker 的启动流程比较类似，这里我们先简单介绍一个流程，以便后面梳理时能更好地理解。

这里涉及几个类：NettyRpcEnv、Dispatcher、EndpointData 和 Inbox。
其调用过程是：
1. 初始化 NettyRpcEnv.
2. 执行步骤 1 时，需要初始化其成员 Dispatcher.
3. 初始化 NettyRpcEnv 完成后，调用其方法 setupEndpoint
4. 调用方法里，会调用 Dispatcher 的方法 registerRpcEndpoint
5. 调用 Dispatcher 的方法 registerRpcEndpoint 时，会初始化 EndpointData.
6. 初始化 EndpointData 时，会初始化 Inbox
7. 初始化 Inbox 时，首先会把 OnStart 事件放到 Inbox 的消息队列 messages 里；
从而后面触发其方法 process 方法时，会首先执行 EndpointData 对应节点 OnStart 事件

## 启动 spark Master 的流程

启动 spark Master 的流程，在这里我们会详细分析，这个过程，尤其是消息处理的过程，上面已经简单地介绍过了，我们在这里只是较为详细地说一下

### Master 对象的 main 方法

上面我们提到，启动 spark Master 时，是脚本 start-daemon.sh 启动了类 org.apache.spark.deploy.master.Master. 由于 scala 语言的特殊性（这里不讲了），
我们知道，这里是执行了 Master 这个 object 的 main 方法。

```scala
private[deploy] object Master extends Logging {
  val SYSTEM_NAME = "sparkMaster"
  val ENDPOINT_NAME = "Master"
 
  # 从里这开始启动，初始化 SparkConf，并通过方法 startRpcEnvAndEndpoint 创建 rpvEnv，
  # 这个 rpvEnv 是个扩展了抽象类 RpcEnv 的类的实例（此处是 NettyRpcEnv, 后面会讲到）
  def main(argStrings: Array[String]) {
    Utils.initDaemon(log)
    val conf = new SparkConf
    val args = new MasterArguments(argStrings, conf)
    val (rpcEnv, _, _) = startRpcEnvAndEndpoint(args.host, args.port, args.webUiPort, conf)
    rpcEnv.awaitTermination()
  }

  /** Start the Master and return a three tuple of: */
  /**   (1) The Master RpcEnv */
  /**   (2) The web UI bound port */
  /**   (3) The REST server bound port, if any */
  /** 这里启动 Master，并返回 Master 所在的 rpcEnv，至于 webUIPort 和 restPort，这里跳过了 */
  def startRpcEnvAndEndpoint(
      host: String,
      port: Int,
      webUiPort: Int,
      conf: SparkConf): (RpcEnv, Int, Option[Int]) = {
    val securityMgr = new SecurityManager(conf)
    /** 创建 rpvEnv 对象，联系前面提到的流程，知道这里面初始化了 NettyRpcEnv 和 Dispatcher，后面会分析到 */
    val rpcEnv = RpcEnv.create(SYSTEM_NAME, host, port, conf, securityMgr)
    /** 用 rpcEnv 初始化一个 master，并返回其 endpoint, 把 OnStart 事件放到消息队列的方法就是这里 */
    val masterEndpoint = rpcEnv.setupEndpoint(ENDPOINT_NAME,
      new Master(rpcEnv, rpcEnv.address, webUiPort, securityMgr, conf))
    val portsResponse = masterEndpoint.askSync[BoundPortsResponse](BoundPortsRequest)
    (rpcEnv, portsResponse.webUIPort, portsResponse.restPort)
  }
}
```

### NettyRpcEnvFactory 类的 create 方法

这里要说明一下，在 Master 的 main 方法中，调用了 RpcEnv 的 create 方法，这个方法内部，创建了 NettyRpcEnvFactory 对象，
并调用这个对象的 create 方法，如下：

```scala
/** A RpcEnv implementation must have a [[RpcEnvFactory]] implementation with an empty constructor */
/** so that it can be created via Reflection. */
private[spark] object RpcEnv {

  def create(
      name: String,
      host: String,
      port: Int,
      conf: SparkConf,
      securityManager: SecurityManager,
      clientMode: Boolean = false): RpcEnv = {
    create(name, host, host, port, conf, securityManager, clientMode)
  }

  def create(
      name: String,
      bindAddress: String,
      advertiseAddress: String,
      port: Int,
      conf: SparkConf,
      securityManager: SecurityManager,
      clientMode: Boolean): RpcEnv = {
    val config = RpcEnvConfig(conf, name, bindAddress, advertiseAddress, port, securityManager,
      clientMode)
    new NettyRpcEnvFactory().create(config)
  }
}
```

对于 NettyRpcEnvFactory 的 create 方法，这里会创建类 NettyRpcEnv 的对象，如下：

```scala
private[rpc] class NettyRpcEnvFactory extends RpcEnvFactory with Logging {

  def create(config: RpcEnvConfig): RpcEnv = {
    val sparkConf = config.conf
    /** Use JavaSerializerInstance in multiple threads is safe. However, if we plan to support */
    /** KryoSerializer in future, we have to use ThreadLocal to store SerializerInstance */
    val javaSerializerInstance =
      new JavaSerializer(sparkConf).newInstance().asInstanceOf[JavaSerializerInstance]
    /** 这里创建 NettyRpcEnv 对象, 这个类扩展了 RpcEnv 这个抽象类 */
    val nettyEnv =
      new NettyRpcEnv(sparkConf, javaSerializerInstance, config.advertiseAddress,
        config.securityManager)
    if (!config.clientMode) {
      val startNettyRpcEnv: Int => (NettyRpcEnv, Int) = { actualPort =>
        nettyEnv.startServer(config.bindAddress, actualPort)
        (nettyEnv, nettyEnv.address.port)
      }
      try {
        Utils.startServiceOnPort(config.port, startNettyRpcEnv, sparkConf, config.name)._1
      } catch {
        case NonFatal(e) =>
          nettyEnv.shutdown()
          throw e
      }
    }
    nettyEnv
  }
}
```

### NettyRpcEnv 类的初始化

NettyRpcEnv 扩展了抽象类 RpcEnv, 其中对本文比较重要的的成员变量是：
dispatcher、transportContext、outboxes.

```scala
private[netty] class NettyRpcEnv(
    val conf: SparkConf,
    javaSerializerInstance: JavaSerializerInstance,
    host: String,
    securityManager: SecurityManager) extends RpcEnv(conf) with Logging {

  private[netty] val transportConf = SparkTransportConf.fromSparkConf(
    conf.clone.set("spark.rpc.io.numConnectionsPerPeer", "1"),
    "rpc",
    conf.getInt("spark.rpc.io.threads", 0))
  
  /** dispatcher 的成员方法里初始化了 EndpointData, 并在里面初始化 Inbox */
  private val dispatcher: Dispatcher = new Dispatcher(this)

  private val streamManager = new NettyStreamManager(this)

  private val transportContext = new TransportContext(transportConf,
    new NettyRpcHandler(dispatcher, this, streamManager))

  private def createClientBootstraps(): java.util.List[TransportClientBootstrap] = {
    if (securityManager.isAuthenticationEnabled()) {
      java.util.Arrays.asList(new AuthClientBootstrap(transportConf,
        securityManager.getSaslUser(), securityManager))
    } else {
      java.util.Collections.emptyList[TransportClientBootstrap]
    }
  }

  private val clientFactory = transportContext.createClientFactory(createClientBootstraps())

  /** A separate client factory for file downloads. This avoids using the same RPC handler as */
  /** the main RPC context, so that events caused by these clients are kept isolated from the */
  /** main RPC traffic. */
  /** */
  /** It also allows for different configuration of certain properties, such as the number of */
  /** connections per peer. */
  @volatile private var fileDownloadFactory: TransportClientFactory = _

  val timeoutScheduler = ThreadUtils.newDaemonSingleThreadScheduledExecutor("netty-rpc-env-timeout")

  /** Because TransportClientFactory.createClient is blocking, we need to run it in this thread pool */
  /** to implement non-blocking send/ask. */
  /** TODO: a non-blocking TransportClientFactory.createClient in future */
  private[netty] val clientConnectionExecutor = ThreadUtils.newDaemonCachedThreadPool(
    "netty-rpc-connection",
    conf.getInt("spark.rpc.connect.threads", 64))

  @volatile private var server: TransportServer = _

  private val stopped = new AtomicBoolean(false)

  /** A map for [[RpcAddress]] and [[Outbox]]. When we are connecting to a remote [[RpcAddress]], */
  /** we just put messages to its [[Outbox]] to implement a non-blocking `send` method. */
  private val outboxes = new ConcurrentHashMap[RpcAddress, Outbox]()

  /** 跳过成员方法 */
  ......
}
```

### Dispatcher 类对消息的处理逻辑

这里把 Dispatcher 整个类放过来，是因为这个类的逻辑，尤其是对消息的处理很重要

```scala
/** A message dispatcher, responsible for routing RPC messages to the appropriate endpoint(s). */
private[netty] class Dispatcher(nettyEnv: NettyRpcEnv) extends Logging {

  private class EndpointData(
      val name: String,
      val endpoint: RpcEndpoint,
      val ref: NettyRpcEndpointRef) {
    /** Inbox 在初始化时会首先将 OnStart 事件放到其消息队列中 */
    val inbox = new Inbox(ref, endpoint)
  }

  private val endpoints: ConcurrentMap[String, EndpointData] =
    new ConcurrentHashMap[String, EndpointData]
  private val endpointRefs: ConcurrentMap[RpcEndpoint, RpcEndpointRef] =
    new ConcurrentHashMap[RpcEndpoint, RpcEndpointRef]

  /** Track the receivers whose inboxes may contain messages. */
  private val receivers = new LinkedBlockingQueue[EndpointData]

  /** True if the dispatcher has been stopped. Once stopped, all messages posted will be bounced */
  /** immediately. */
  @GuardedBy("this")
  private var stopped = false

  def registerRpcEndpoint(name: String, endpoint: RpcEndpoint): NettyRpcEndpointRef = {
    val addr = RpcEndpointAddress(nettyEnv.address, name)
    val endpointRef = new NettyRpcEndpointRef(nettyEnv.conf, addr, nettyEnv)
    synchronized {
      if (stopped) {
        throw new IllegalStateException("RpcEnv has been stopped")
      }
      /** 注册 RpcEndpoint 时初始化 EndpointData, 也就是在这里初始化了 Inbox, 将 OnStart 加入消息队列 */
      if (endpoints.putIfAbsent(name, new EndpointData(name, endpoint, endpointRef)) != null) {
        throw new IllegalArgumentException(s"There is already an RpcEndpoint called $name")
      }
      val data = endpoints.get(name)
      endpointRefs.put(data.endpoint, data.ref)
      /** OnStart 在 endpoints 中，需要获取出来，receivers 里要记录的，是可能包含消息的 endpoint */
      receivers.offer(data)  // for the OnStart message
    }
    endpointRef
  }

  def getRpcEndpointRef(endpoint: RpcEndpoint): RpcEndpointRef = endpointRefs.get(endpoint)

  def removeRpcEndpointRef(endpoint: RpcEndpoint): Unit = endpointRefs.remove(endpoint)

  /** Should be idempotent */
  private def unregisterRpcEndpoint(name: String): Unit = {
    val data = endpoints.remove(name)
    if (data != null) {
      data.inbox.stop()
      receivers.offer(data)  // for the OnStop message
    }
    /** Don't clean `endpointRefs` here because it's possible that some messages are being processed */
    /** now and they can use `getRpcEndpointRef`. So `endpointRefs` will be cleaned in Inbox via */
    /**`removeRpcEndpointRef`. */
  }

  def stop(rpcEndpointRef: RpcEndpointRef): Unit = {
    synchronized {
      if (stopped) {
        /** This endpoint will be stopped by Dispatcher.stop() method. */
        return
      }
      unregisterRpcEndpoint(rpcEndpointRef.name)
    }
  }

  /** Send a message to all registered [[RpcEndpoint]]s in this process. */
  /** */
  /** This can be used to make network events known to all end points (e.g. "a new node connected"). */
  def postToAll(message: InboxMessage): Unit = {
    val iter = endpoints.keySet().iterator()
    while (iter.hasNext) {
      val name = iter.next
      postMessage(name, message, (e) => logWarning(s"Message $message dropped. ${e.getMessage}"))
    }
  }

  /** Posts a message sent by a remote endpoint. */
  def postRemoteMessage(message: RequestMessage, callback: RpcResponseCallback): Unit = {
    val rpcCallContext =
      new RemoteNettyRpcCallContext(nettyEnv, callback, message.senderAddress)
    val rpcMessage = RpcMessage(message.senderAddress, message.content, rpcCallContext)
    postMessage(message.receiver.name, rpcMessage, (e) => callback.onFailure(e))
  }

  /** Posts a message sent by a local endpoint. */
  def postLocalMessage(message: RequestMessage, p: Promise[Any]): Unit = {
    val rpcCallContext =
      new LocalNettyRpcCallContext(message.senderAddress, p)
    val rpcMessage = RpcMessage(message.senderAddress, message.content, rpcCallContext)
    postMessage(message.receiver.name, rpcMessage, (e) => p.tryFailure(e))
  }

  /** Posts a one-way message. */
  def postOneWayMessage(message: RequestMessage): Unit = {
    postMessage(message.receiver.name, OneWayMessage(message.senderAddress, message.content),
      (e) => throw e)
  }

  /** Posts a message to a specific endpoint. */
  /** */
  /** @param endpointName name of the endpoint. */
  /** @param message the message to post */
  /** @param callbackIfStopped callback function if the endpoint is stopped. */
  private def postMessage(
      endpointName: String,
      message: InboxMessage,
      callbackIfStopped: (Exception) => Unit): Unit = {
    val error = synchronized {
      val data = endpoints.get(endpointName)
      if (stopped) {
        Some(new RpcEnvStoppedException())
      } else if (data == null) {
        Some(new SparkException(s"Could not find $endpointName."))
      } else {
        data.inbox.post(message)
        receivers.offer(data)
        None
      }
    }
    /** We don't need to call `onStop` in the `synchronized` block */
    error.foreach(callbackIfStopped)
  }

  def stop(): Unit = {
    synchronized {
      if (stopped) {
        return
      }
      stopped = true
    }
    /** Stop all endpoints. This will queue all endpoints for processing by the message loops. */
    endpoints.keySet().asScala.foreach(unregisterRpcEndpoint)
    /** Enqueue a message that tells the message loops to stop. */
    receivers.offer(PoisonPill)
    threadpool.shutdown()
  }

  def awaitTermination(): Unit = {
    threadpool.awaitTermination(Long.MaxValue, TimeUnit.MILLISECONDS)
  }

  /** Return if the endpoint exists */
  def verify(name: String): Boolean = {
    endpoints.containsKey(name)
  }

  /** Thread pool used for dispatching messages. */
  /** 处理的消息的线程池，这里是处理消息逻辑最关键的地方！！！ */
  private val threadpool: ThreadPoolExecutor = {
    val numThreads = nettyEnv.conf.getInt("spark.rpc.netty.dispatcher.numThreads",
      math.max(2, Runtime.getRuntime.availableProcessors()))
    /** 起多个线程，每个线程都可以用来处理消息 */
    val pool = ThreadUtils.newDaemonFixedThreadPool(numThreads, "dispatcher-event-loop")
    for (i <- 0 until numThreads) {
      pool.execute(new MessageLoop)
    }
    pool
  }

  /** Message loop used for dispatching messages. */
  /** Message loop 实现了 Runnable 接口，是一个线程 */
  private class MessageLoop extends Runnable {
    override def run(): Unit = {
      try {
        /** 由于循环的条件恒为 true, 只有当消息是 PoisonPill 时，否则循环永不退出 */
        while (true) {
          try {
            /** 我们知道 receivers 是可能包含了消息的 inbox 所属的 endpoint, 这里取出来 endpoint */
            val data = receivers.take()
            if (data == PoisonPill) {
              // Put PoisonPill back so that other MessageLoops can see it.
              receivers.offer(PoisonPill)
              return
            }
            /** 触发消息的执行!!! */
            data.inbox.process(Dispatcher.this)
          } catch {
            case NonFatal(e) => logError(e.getMessage, e)
          }
        }
      } catch {
        case ie: InterruptedException => // exit
      }
    }
  }

  /** A poison endpoint that indicates MessageLoop should exit its message loop. */
  private val PoisonPill = new EndpointData(null, null, null)
}
```

### Inbox 的初始化和 process 方法

前面的内容看过后，相信对 Inbox 的作用有了一定的了解，这里我们将看到，Inbox 是真正地调用 endpoint 的消息来触发远程事件的执行。

```scala
/** An inbox that stores messages for an [[RpcEndpoint]] and posts messages to it thread-safely. */
private[netty] class Inbox(
    val endpointRef: NettyRpcEndpointRef,
    val endpoint: RpcEndpoint)
  extends Logging {

  inbox =>  // Give this an alias so we can use it more clearly in closures.

  @GuardedBy("this")
  protected val messages = new java.util.LinkedList[InboxMessage]()

  /** True if the inbox (and its associated endpoint) is stopped. */
  @GuardedBy("this")
  private var stopped = false

  /** Allow multiple threads to process messages at the same time. */
  @GuardedBy("this")
  private var enableConcurrent = false

  /** The number of threads processing messages for this inbox. */
  @GuardedBy("this")
  private var numActiveThreads = 0

  /** OnStart should be the first message to process */
  /** 初始化的时候，首先将 OnStart 消息加入消息队列中, 则只要第一次处理 messages, 即第一次调用 process，一定会先触发 OnStart 事件 */
  inbox.synchronized {
    messages.add(OnStart)
  }

  /** Process stored messages. */
  def process(dispatcher: Dispatcher): Unit = {
    var message: InboxMessage = null
    inbox.synchronized {
      if (!enableConcurrent && numActiveThreads != 0) {
        return
      }
      message = messages.poll()
      if (message != null) {
        numActiveThreads += 1
      } else {
        return
      }
    }
    while (true) {
      safelyCall(endpoint) {
        message match {
          case RpcMessage(_sender, content, context) =>
            try {
              /** 调用 endpoint 端的 receiveAndReply 函数，适用于需要响应的消息 */
              endpoint.receiveAndReply(context).applyOrElse[Any, Unit](content, { msg =>
                throw new SparkException(s"Unsupported message $message from ${_sender}")
              })
            } catch {
              case NonFatal(e) =>
                context.sendFailure(e)
                /** Throw the exception -- this exception will be caught by the safelyCall function. */
                /** The endpoint's onError function will be called. */
                throw e
            }

          case OneWayMessage(_sender, content) =>
            /** 调用 endpoint 端的 receive 函数，适用于不需要响应的消息 */
            endpoint.receive.applyOrElse[Any, Unit](content, { msg =>
              throw new SparkException(s"Unsupported message $message from ${_sender}")
            })

          case OnStart =>
            /** 当消息类型是 OnStart 时，触发 endpoint 端的 onStart 函数，那些 onStart 函数的调用很多是从这里发起的 */
            endpoint.onStart()
            if (!endpoint.isInstanceOf[ThreadSafeRpcEndpoint]) {
              inbox.synchronized {
                if (!stopped) {
                  enableConcurrent = true
                }
              }
            }

          case OnStop =>
            val activeThreads = inbox.synchronized { inbox.numActiveThreads }
            assert(activeThreads == 1,
              s"There should be only a single active thread but found $activeThreads threads.")
            dispatcher.removeRpcEndpointRef(endpoint)
            /** 当消息类型是 OnStop 时，触发 endpoint 端的 onStop 函数，那些 onStop 函数的调用很多是从这里发起的 */
            endpoint.onStop()
            assert(isEmpty, "OnStop should be the last message")

          case RemoteProcessConnected(remoteAddress) =>
            endpoint.onConnected(remoteAddress)

          case RemoteProcessDisconnected(remoteAddress) =>
            endpoint.onDisconnected(remoteAddress)

          case RemoteProcessConnectionError(cause, remoteAddress) =>
            endpoint.onNetworkError(cause, remoteAddress)
        }
      }

      inbox.synchronized {
        /** "enableConcurrent" will be set to false after `onStop` is called, so we should check it */
        /** every time. */
        if (!enableConcurrent && numActiveThreads != 1) {
          /** If we are not the only one worker, exit */
          numActiveThreads -= 1
          return
        }
        message = messages.poll()
        if (message == null) {
          numActiveThreads -= 1
          return
        }
      }
    }
  }

  /** 跳过一些成员方法 */
}
```

### 调用 spark Master 的 onStart 方法

在 inbox 中调用了 master 的 onStart 方法后，master 会根据 RECOVERY_MODE 确定主节点选取客户端 leaderElectionAgent，
这里我们通常选择 zk，不过我有一个疑问：从 Master 的方法 electedLeader 向上追溯，追溯到类
ZooKeeperLeaderElectionAgent 的方法 isLeader 后，就再也无法向上找到 master 主节点竞争的函数调用了，那是哪里产生而触发的
主节点竞争呢？zk 自己做的吗？

```scala
override def onStart(): Unit = {
  logInfo("Starting Spark master at " + masterUrl)
  logInfo(s"Running Spark version ${org.apache.spark.SPARK_VERSION}")
  webUi = new MasterWebUI(this, webUiPort)
  webUi.bind()
  masterWebUiUrl = "http://" + masterPublicAddress + ":" + webUi.boundPort
  if (reverseProxy) {
    masterWebUiUrl = conf.get("spark.ui.reverseProxyUrl", masterWebUiUrl)
    logInfo(s"Spark Master is acting as a reverse proxy. Master, Workers and " +
     s"Applications UIs are available at $masterWebUiUrl")
  }
  checkForWorkerTimeOutTask = forwardMessageThread.scheduleAtFixedRate(new Runnable {
    override def run(): Unit = Utils.tryLogNonFatalError {
      self.send(CheckForWorkerTimeOut)
    }
  }, 0, WORKER_TIMEOUT_MS, TimeUnit.MILLISECONDS)

  if (restServerEnabled) {
    val port = conf.getInt("spark.master.rest.port", 6066)
    restServer = Some(new StandaloneRestServer(address.host, port, conf, self, masterUrl))
  }
  restServerBoundPort = restServer.map(_.start())

  masterMetricsSystem.registerSource(masterSource)
  masterMetricsSystem.start()
  applicationMetricsSystem.start()
  // Attach the master and app metrics servlet handler to the web ui after the metrics systems are
  // started.
  masterMetricsSystem.getServletHandlers.foreach(webUi.attachHandler)
  applicationMetricsSystem.getServletHandlers.foreach(webUi.attachHandler)

  val serializer = new JavaSerializer(conf)
  val (persistenceEngine_, leaderElectionAgent_) = RECOVERY_MODE match {
    case "ZOOKEEPER" =>
      logInfo("Persisting recovery state to ZooKeeper")
      val zkFactory =
        new ZooKeeperRecoveryModeFactory(conf, serializer)
      (zkFactory.createPersistenceEngine(), zkFactory.createLeaderElectionAgent(this))
    case "FILESYSTEM" =>
      val fsFactory =
        new FileSystemRecoveryModeFactory(conf, serializer)
      (fsFactory.createPersistenceEngine(), fsFactory.createLeaderElectionAgent(this))
    case "CUSTOM" =>
      val clazz = Utils.classForName(conf.get("spark.deploy.recoveryMode.factory"))
      val factory = clazz.getConstructor(classOf[SparkConf], classOf[Serializer])
        .newInstance(conf, serializer)
        .asInstanceOf[StandaloneRecoveryModeFactory]
      (factory.createPersistenceEngine(), factory.createLeaderElectionAgent(this))
    case _ =>
      (new BlackHolePersistenceEngine(), new MonarchyLeaderAgent(this))
  }
  persistenceEngine = persistenceEngine_
  leaderElectionAgent = leaderElectionAgent_
}
```

总之，到这里，master 节点就起来了，其它要做的事情，是 master 起来之后要做的，我们后面再分析。

## spark Worker 启动流程

启动 spark Worker 的流程，在这里我们会详细分析，不过启动过程中涉及消息传递和处理的过程，和 master 启动完全相同，我们就不再赘述了，重点关注
worker 起来之后，与 master 做的一些交互。

### Worker 的 main 方法

TODO 未完待续
