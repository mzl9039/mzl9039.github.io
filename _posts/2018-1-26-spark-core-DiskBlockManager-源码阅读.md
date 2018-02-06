---
layout:     post
title:      "Spark DiskBlockManager源码阅读"
subtitle:   ""
date:       2018-1-26 22:00:05
author:     mzl
catalog:    true
tags:
    - spark
    - DiskBlockManager
---

# DiskBlockManager
DiskBlockManager 是用来创建和控制逻辑上的映射关系（逻辑上的 block 和物理盘上的 locations）。
一个 block 被映射为一个文件 File，这个文件的文件名由指定的 BlockId.name 指定。
```scala
// 类的定义和类的属性成员

private[spark] class DiskBlockManager(conf: SparkConf, deleteFilesOnStop: Boolean) extends Logging {
  // 从配置获取每个目录下的文件数

  private[spark] val subDirsPerLocalDir = conf.getInt("spark.diskStore.subDirectories", 64)
  
  // 根据配置创建本地目录 localDirs，这些目录为 spark.local.dir 的配置项的内容。然后在这些目录里，创建很多

  // 个子目录 subDirs，将来我们会把文件 hash 到这些子目录里（这是为了避免在 top level 有非常大的 inodes）

  // 疑问：这里的 inode 指的是硬盘的 inode? 不太明白，猜测是一个目录下有过多的文件时，会产生非常大的 inode

  /* Create one local directory for each path mentioned in spark.local.dir; then, inside this
   * directory, create multiple subdirectories that we will hash files into, in order to avoid
   * having really large inodes at the top level. */
  private[spark] val localDirs: Array[File] = createLocalDirs(conf)
  if (localDirs.isEmpty) {
    logError("Failed to create any local dir.")
    System.exit(ExecutorExitCode.DISK_STORE_FAILED_TO_CREATE_DIR)
  }
  // 上面注释中提到的子目录，子目录本身是不可变的，但子目录（作为一个数组）的每个数组项是可变的。

  // 每个数组项都被这个项 lock (后面的代码里使用了 synchronize)

  // The content of subDirs is immutable but the content of subDirs(i) is mutable. And the content

  // of subDirs(i) is protected by the lock of subDirs(i)

  private val subDirs = Array.fill(localDirs.length)(new Array[File](subDirsPerLocalDir))

  private val shutdownHook = addShutdownHook()
  // 此处省略了很多成员方法，后面讲
}
```
## getFile

用来查找文件（根据 hash 来确定它在哪个本地子目录）
```scala
  /** Looks up a file by hashing it into one of our local subdirectories. */
  // 这个方法要和 ExternalShuffleBlockResolver 类里的 getFile 方法逻辑上保持一致.

  // 简单看了下，两个方法实现的是同样的功能，所以要保持一致。因为两个方法要保持相同的功能。

  // This method should be kept in sync with

  // org.apache.spark.network.shuffle.ExternalShuffleBlockResolver#getFile().

  def getFile(filename: String): File = {
    // Figure out which local directory it hashes to, and which subdirectory in that

    // 找到文件被 hash 到了哪个本地目录，以及它在哪个子目录

    // 这里的 hash 几乎为 filename 的 hashCode。而寻找本地目录和子目录的过程，都是 %

    val hash = Utils.nonNegativeHash(filename)
    val dirId = hash % localDirs.length
    val subDirId = (hash / localDirs.length) % subDirsPerLocalDir

    // Create the subdirectory if it doesn't already exist

    // 这里应了上面的话：每个数组项都被这个项 lock。另外，这里会在文件不存在的时候创建

    val subDir = subDirs(dirId).synchronized {
      val old = subDirs(dirId)(subDirId)
      if (old != null) {
        old
      } else {
        val newDir = new File(localDirs(dirId), "%02x".format(subDirId))
        if (!newDir.exists() && !newDir.mkdir()) {
          throw new IOException(s"Failed to create local dir in $newDir.")
        }
        subDirs(dirId)(subDirId) = newDir
        newDir
      }
    }

    new File(subDir, filename)
  }

  def getFile(blockId: BlockId): File = getFile(blockId.name)
```

## getAllFiles

列出当前 disk manager 下的所有文件。
```scala
  /** Check if disk block manager has a block. */
  def containsBlock(blockId: BlockId): Boolean = {
    getFile(blockId.name).exists()
  }

  /** List all the files currently stored on disk by the disk manager. */
  /** 注意这里的 dir.clone 的思想。*/
  def getAllFiles(): Seq[File] = {
    // Get all the files inside the array of array of directories

    subDirs.flatMap { dir =>
      dir.synchronized {
        // Copy the content of dir because it may be modified in other threads

        dir.clone()
      }
    }.filter(_ != null).flatMap { dir =>
      val files = dir.listFiles()
      if (files != null) files else Seq.empty
    }
  }

  /** List all the blocks currently stored on disk by the disk manager. */
  def getAllBlocks(): Seq[BlockId] = {
    getAllFiles().map(f => BlockId(f.getName))
  }
```

## createTempLocalBlock 和 createTempShuffleBlock

生成一个唯一的 block id 和文件, 用来存储 Local 或 Shuffle 的中间结果
```scala
  /** Produces a unique block id and File suitable for storing local intermediate results. */
  def createTempLocalBlock(): (TempLocalBlockId, File) = {
    var blockId = new TempLocalBlockId(UUID.randomUUID())
    while (getFile(blockId).exists()) {
      blockId = new TempLocalBlockId(UUID.randomUUID())
    }
    (blockId, getFile(blockId))
  }

  /** Produces a unique block id and File suitable for storing shuffled intermediate results. */
  def createTempShuffleBlock(): (TempShuffleBlockId, File) = {
    var blockId = new TempShuffleBlockId(UUID.randomUUID())
    while (getFile(blockId).exists()) {
      blockId = new TempShuffleBlockId(UUID.randomUUID())
    }
    (blockId, getFile(blockId))
  }
```

## createLocalDirs

为要存储的 block data 创建本地目录。这些目录在配置好的本地目录中，且当使用外部 shuffle 服务时，
这些目录在 JVM 退出时不会被删除。
```scala
  /**
    * Create local directories for storing block data. These directories are
    * located inside configured local directories and won't
    * be deleted on JVM exit when using the external shuffle service.
    */
  private def createLocalDirs(conf: SparkConf): Array[File] = {
    Utils.getConfiguredLocalDirs(conf).flatMap { rootDir =>
      try {
        // 本地目录，名称中带 blockmgr 的目录在这里创建

        val localDir = Utils.createDirectory(rootDir, "blockmgr")
        logInfo(s"Created local directory at $localDir")
        Some(localDir)
      } catch {
        case e: IOException =>
          logError(s"Failed to create local dir in $rootDir. Ignoring this directory.", e)
          None
      }
    }
  }
```

## doStop

程序结束时，执行 doStop 删除本地目录。这里的 addShutdownHook 通过添加 doStop 的 hook 调用 doStop，而 stop 则是直接调用了 doStop
```scala
  private def addShutdownHook(): AnyRef = {
    logDebug("Adding shutdown hook") // force eager creation of logger
    // 通过添加 shutdownHook 调用 doStop

    ShutdownHookManager.addShutdownHook(ShutdownHookManager.TEMP_DIR_SHUTDOWN_PRIORITY + 1) { () =>
      logInfo("Shutdown hook called")
      DiskBlockManager.this.doStop()
    }
  }

  /** Cleanup local dirs and stop shuffle sender. */
  private[spark] def stop(): Unit = {
    // Remove the shutdown hook.  It causes memory leaks if we leave it around.

    // 先删除 shutdown hook，否则可能会有内存泄漏，对 hook 的分析要看 ShutdownHookManager

    try {
      ShutdownHookManager.removeShutdownHook(shutdownHook)
    } catch {
      case e: Exception =>
        logError(s"Exception while removing shutdown hook.", e)
    }
    doStop()
  }

  private def doStop(): Unit = {
    // deleteFilesOnStop 是传进来的参数，在 BlockManager 中被初始化，跟 external shuffle service(配置项："spark.shuffle.service.enabled") 有关

    if (deleteFilesOnStop) {
      localDirs.foreach { localDir =>
        if (localDir.isDirectory && localDir.exists()) {
          try {
            if (!ShutdownHookManager.hasRootAsShutdownDeleteDir(localDir)) {
              Utils.deleteRecursively(localDir)
            }
          } catch {
            case e: Exception =>
              logError(s"Exception while deleting local spark dir: $localDir", e)
          }
        }
      }
    }
  }
```
