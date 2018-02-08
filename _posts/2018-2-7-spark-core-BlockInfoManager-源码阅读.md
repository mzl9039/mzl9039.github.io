---
layout:     post
title:      "Spark BlockInfoManager源码阅读"
subtitle:   ""
date:       2018-2-7 22:00:05
author:     mzl
catalog:    true
tags:
    - spark
    - BlockInfo
    - BlockInfoManager
---

# BlockInfoManager.scala 这个文件包括两个类：BlockInfo 和 BlockInfoManager
## BlockInfo

追踪某个 Block 的元信息。
这个类的实例并不是线程安全的，由类 BlockInfoManager 中的锁保护。
```scala
/**
 * 这个类有三个参数，有三个属性成员。
 * @param level the block's storage level. This is the requested persistence level, not the
 *              effective storage level of the block (i.e. if this is MEMORY_AND_DISK, then this
 *              does not imply that the block is actually resident in memory).
 *              是 block 的 storage level. 这是当前 block 在请求中的持久性级别，而不是真实的存储级别。
 *              （例如如果是 MEMORY_AND_DISK，那这个类并不会去确认当前 block 是否真的在内存中常驻）
 * @param classTag the block's [[ClassTag]], used to select the serializer. 
 *                 这个 block 的类标识，用于选择序列化类
 * @param tellMaster whether state changes for this block should be reported to the master. This
 *                   is true for most blocks, but is false for broadcast blocks.
 *                   这个 block 的状态变化时，是否通知 master. 对大多数 block 来说，这个值通常为 true，
 *                   但对广播 block，这个值为 false。
 * 成员属性有：_size, _readerCount, _writerTask
 *             _size: 表示 block 的大小
 *             _readerCount: 记录 block 被读锁锁的次数（每个读都会加 1 次）
 *             _writerTask: 拥有写锁的 task attempt 的 id。如果写锁被 non-task code 拥有，则为 NON_TASK_WRITER;
 *                          如果没有写锁，则为 NO_WRITER
 */
private[storage] class BlockInfo(
    val level: StorageLevel,
    val classTag: ClassTag[_],
    val tellMaster: Boolean) {

  /**
    * The size of the block (in bytes)
    * 这个 block 的大小
    */
  def size: Long = _size
  def size_=(s: Long): Unit = {
    _size = s
    checkInvariants()
  }
  private[this] var _size: Long = 0

  /**
    * The number of times that this block has been locked for reading.
    */
  def readerCount: Int = _readerCount
  def readerCount_=(c: Int): Unit = {
    _readerCount = c
    checkInvariants()
  }
  private[this] var _readerCount: Int = 0

  /**
    * The task attempt id of the task which currently holds the write lock for this block, or
    * [[BlockInfo.NON_TASK_WRITER]] if the write lock is held by non-task code, or
    * [[BlockInfo.NO_WRITER]] if this block is not locked for writing.
    */
  def writerTask: Long = _writerTask
  def writerTask_=(t: Long): Unit = {
    _writerTask = t
    checkInvariants()
  }
  private[this] var _writerTask: Long = BlockInfo.NO_WRITER

  private def checkInvariants(): Unit = {
    // A block's reader count must be non-negative:
    assert(_readerCount >= 0)
    // A block is either locked for reading or for writing, but not for both at the same time:
    assert(_readerCount == 0 || _writerTask == BlockInfo.NO_WRITER)
  }

  checkInvariants()
}
```
## BlockInfo object

BlockInfo 的 object，定义了 NON_TASK_WRITER 和 NO_WRITER
```scala
private[storage] object BlockInfo {

  /**
    * Special task attempt id constant used to mark a block's write lock as being unlocked.
    * 特殊的 task attempt id 常量，用来标记 block 没有写锁
    */
  val NO_WRITER: Long = -1

  /**
    * Special task attempt id constant used to mark a block's write lock as being held by
    * a non-task thread (e.g. by a driver thread or by unit test code).
    * 特定的 task attempt id 常量，用来标识 block 的写锁被 non-task 线程/code 拥有
    */
  val NON_TASK_WRITER: Long = -1024
}
```
## BlockInfoManager

用于追踪 block 的元数据信息和管理 block 锁的组件。

这个类暴露出来的锁接口是读写锁。每个锁的获得，自动和 running task 相关系，而锁的释放，也自动在
任务结束或失败时释放。

这个类是线程安全的。
```scala
private[storage] class BlockInfoManager extends Logging {

  /* 将 Long 类型重定义为 TaskAttemptId 类型，使用的是 type 关键字 */
  private type TaskAttemptId = Long

  /**
    * Used to look up metadata for individual blocks. Entries are added to this map via an atomic
    * set-if-not-exists operation ([[lockNewBlockForWriting()]]) and are removed
    * by [[removeBlock()]].
    * 用来查找 block 的元数据信息。map 的 entry 在 lockNewBlockForWriting 中自动添加了（set-if-not-exists）
    * 并且在 removeBlock 中被删除。
    * GuardedBy("this") 标签表示 this 是当前属性锁定的时对象
    */
  @GuardedBy("this")
  private[this] val infos = new mutable.HashMap[BlockId, BlockInfo]

  /**
    * Tracks the set of blocks that each task has locked for writing.
    * 每个 task 的写锁已经锁定的 block 的集合
    */
  @GuardedBy("this")
  private[this] val writeLocksByTask =
    new mutable.HashMap[TaskAttemptId, mutable.Set[BlockId]]
      with mutable.MultiMap[TaskAttemptId, BlockId]

  /**
    * Tracks the set of blocks that each task has locked for reading, along with the number of times
    * that a block has been locked (since our read locks are re-entrant).
    * 追踪每个 task 的读锁已经锁定的 block 的集合，以及这个 block 被锁定的次数，一个 task 拥有多个 block 的读锁，都会记录
    */
  @GuardedBy("this")
  private[this] val readLocksByTask =
    new mutable.HashMap[TaskAttemptId, ConcurrentHashMultiset[BlockId]]
}
```
### registerTask 和 currentTaskAttemptId

注册task方法，和获取 currentTaskAttemptId 的方法, 没什么写的，跳过。
```scala
  // ----------------------------------------------------------------------------------------------

  // Initialization for special task attempt ids:
  registerTask(BlockInfo.NON_TASK_WRITER)

  // ----------------------------------------------------------------------------------------------

  /**
    * Called at the start of a task in order to register that task with this [[BlockInfoManager]].
    * This must be called prior to calling any other BlockInfoManager methods from that task.
    */
  def registerTask(taskAttemptId: TaskAttemptId): Unit = synchronized {
    require(!readLocksByTask.contains(taskAttemptId),
      s"Task attempt $taskAttemptId is already registered")
    readLocksByTask(taskAttemptId) = ConcurrentHashMultiset.create()
  }

  /**
    * Returns the current task's task attempt id (which uniquely identifies the task), or
    * [[BlockInfo.NON_TASK_WRITER]] if called by a non-task thread.
    */
  private def currentTaskAttemptId: TaskAttemptId = {
    Option(TaskContext.get()).map(_.taskAttemptId()).getOrElse(BlockInfo.NON_TASK_WRITER)
  }
```

### lockForReading 和 lockForWriting 等方法

这里包括了读锁和写锁的加锁和释放锁的所有方法
```scala
  /**
    * Lock a block for reading and return its metadata.
    * 为了读取block，锁住 block，并返回 block 的元数据信息.
    * If another task has already locked this block for reading, then the read lock will be
    * immediately granted to the calling task and its lock count will be incremented.
    * 如果申请 block 的读锁的时候，读锁已经被另一个 task 锁定，这个读锁会立刻授权给当前的task
    * 并且锁定次数加 1
    * If another task has locked this block for writing, then this call will block until the write
    * lock is released or will return immediately if `blocking = false`.
    * 如果申请 block 的写锁的时候，写锁已经被另一个 task 锁定，如果 blocking = false, 则这个申请会
    * 被阻塞直到原来的写锁被释放；如果 blocking = true, 则会立即返回。
    * A single task can lock a block multiple times for reading, in which case each lock will need
    * to be released separately.
    * 一个 task 可以对一个 block 有多个读锁，这种情况下，每个读锁都需要单独释放。
    * @param blockId the block to lock.
    * @param blocking if true (default), this call will block until the lock is acquired. If false,
    *                 this call will return immediately if the lock acquisition fails.
    * @return None if the block did not exist or was removed (in which case no lock is held), or
    *         Some(BlockInfo) (in which case the block is locked for reading).
    */
  def lockForReading(
      blockId: BlockId,
      blocking: Boolean = true): Option[BlockInfo] = synchronized {
    logTrace(s"Task $currentTaskAttemptId trying to acquire read lock for $blockId")
    do {
      infos.get(blockId) match {
        case None => return None
        case Some(info) =>
          if (info.writerTask == BlockInfo.NO_WRITER) {
            info.readerCount += 1
            readLocksByTask(currentTaskAttemptId).add(blockId)
            logTrace(s"Task $currentTaskAttemptId acquired read lock for $blockId")
            return Some(info)
          }
      }
      if (blocking) {
        wait()
      }
    } while (blocking)
    None
  }

  /**
    * Lock a block for writing and return its metadata.
    * 为一个 block 添加写锁，并返回它的元数据信息
    * If another task has already locked this block for either reading or writing, then this call
    * will block until the other locks are released or will return immediately if `blocking = false`.
    * 当想读或写一个 block 时,另一个 task 已经锁住了这个 block，则这个返回会根据 blocking 参数确定
    * 返回 None 还是阻塞等待返回写锁
    * @param blockId the block to lock.
    * @param blocking if true (default), this call will block until the lock is acquired. If false,
    *                 this call will return immediately if the lock acquisition fails.
    * @return None if the block did not exist or was removed (in which case no lock is held), or
    *         Some(BlockInfo) (in which case the block is locked for writing).
    */
  def lockForWriting(
      blockId: BlockId,
      blocking: Boolean = true): Option[BlockInfo] = synchronized {
    logTrace(s"Task $currentTaskAttemptId trying to acquire write lock for $blockId")
    do {
      infos.get(blockId) match {
        case None => return None
        case Some(info) =>
          if (info.writerTask == BlockInfo.NO_WRITER && info.readerCount == 0) {
            info.writerTask = currentTaskAttemptId
            writeLocksByTask.addBinding(currentTaskAttemptId, blockId)
            logTrace(s"Task $currentTaskAttemptId acquired write lock for $blockId")
            return Some(info)
          }
      }
      if (blocking) {
        wait()
      }
    } while (blocking)
    None
  }

  /**
    * Throws an exception if the current task does not hold a write lock on the given block.
    * Otherwise, returns the block's BlockInfo.
    */
  def assertBlockIsLockedForWriting(blockId: BlockId): BlockInfo = synchronized {
    infos.get(blockId) match {
      case Some(info) =>
        if (info.writerTask != currentTaskAttemptId) {
          throw new SparkException(
            s"Task $currentTaskAttemptId has not locked block $blockId for writing")
        } else {
          info
        }
      case None =>
        throw new SparkException(s"Block $blockId does not exist")
    }
  }

  /**
    * Get a block's metadata without acquiring any locks. This method is only exposed for use by
    * [[BlockManager.getStatus()]] and should not be called by other code outside of this class.
    */
  private[storage] def get(blockId: BlockId): Option[BlockInfo] = synchronized {
    infos.get(blockId)
  }

  /**
    * Downgrades an exclusive write lock to a shared read lock.
    * 从一个高级的写锁降级为一个共享的读锁
    */
  def downgradeLock(blockId: BlockId): Unit = synchronized {
    logTrace(s"Task $currentTaskAttemptId downgrading write lock for $blockId")
    val info = get(blockId).get
    require(info.writerTask == currentTaskAttemptId,
      s"Task $currentTaskAttemptId tried to downgrade a write lock that it does not hold on" +
        s" block $blockId")
    unlock(blockId)
    val lockOutcome = lockForReading(blockId, blocking = false)
    assert(lockOutcome.isDefined)
  }

  /**
    * Release a lock on the given block.
    * In case a TaskContext is not propagated properly to all child threads for the task, we fail to
    * get the TID from TaskContext, so we have to explicitly pass the TID value to release the lock.
    * 在给定的 block 上释放锁.
    * 为了防止 TaskContext 没有合理的把 task 通知到所有的子线程，我们没有成功
    * 地获取 TID，我们必须明确地传递这个 TID 的值来释放这个锁
    * See SPARK-18406 for more discussion of this issue.
    * https://github.com/apache/spark/pull/18076 
    */
  def unlock(blockId: BlockId, taskAttemptId: Option[TaskAttemptId] = None): Unit = synchronized {
    val taskId = taskAttemptId.getOrElse(currentTaskAttemptId)
    logTrace(s"Task $taskId releasing lock for $blockId")
    val info = get(blockId).getOrElse {
      throw new IllegalStateException(s"Block $blockId not found")
    }
    if (info.writerTask != BlockInfo.NO_WRITER) {
      info.writerTask = BlockInfo.NO_WRITER
      writeLocksByTask.removeBinding(taskId, blockId)
    } else {
      assert(info.readerCount > 0, s"Block $blockId is not locked for reading")
      info.readerCount -= 1
      val countsForTask = readLocksByTask(taskId)
      val newPinCountForTask: Int = countsForTask.remove(blockId, 1) - 1
      assert(newPinCountForTask >= 0,
        s"Task $taskId release lock on block $blockId more times than it acquired it")
    }
    notifyAll()
  }

  /**
    * Attempt to acquire the appropriate lock for writing a new block.
    * 尝试去获取一个新 block 的写锁
    * This enforces the first-writer-wins semantics. If we are the first to write the block,
    * then just go ahead and acquire the write lock. Otherwise, if another thread is already
    * writing the block, then we wait for the write to finish before acquiring the read lock.
    * 这里强制执行 first-writer-wins 语义，即先来的 writer 获取写锁。如果我们是第一个来写这个
    * block，然后只需要获取写锁。否则，如果另一个线程已经在写这个 block，然后我们等待写操作结束，
    * 直到我们获取读锁
    * @return true if the block did not already exist, false otherwise. If this returns false, then
    *         a read lock on the existing block will be held. If this returns true, a write lock on
    *         the new block will be held.
    *         如果这个 block 不存在，则为 true，否则为 false。如果返回 false， 一个在已经
    *         存在的 block 的读锁将被获取到。如果返回 true，一个在新的 block 上的写锁将被获取。
    */
  def lockNewBlockForWriting(
      blockId: BlockId,
      newBlockInfo: BlockInfo): Boolean = synchronized {
    logTrace(s"Task $currentTaskAttemptId trying to put $blockId")
    lockForReading(blockId) match {
      case Some(info) =>
        // Block already exists. This could happen if another thread races with us to compute
        // the same block. In this case, just keep the read lock and return.
        false
      case None =>
        // Block does not yet exist or is removed, so we are free to acquire the write lock
        infos(blockId) = newBlockInfo
        lockForWriting(blockId)
        true
    }
  }

  /**
    * Release all lock held by the given task, clearing that task's pin bookkeeping
    * structures and updating the global pin counts. This method should be called at the
    * end of a task (either by a task completion handler or in `TaskRunner.run()`).
    * 释放一个给定的 task 所拥有的所有的锁，清除保存这个 task 的锁信息的数据结构中，关于
    * 这个 task 的信息，并更新全局 count. 这个方法只应该在 task end 的时候调用。
    * @return the ids of blocks whose pins were released
    */
  def releaseAllLocksForTask(taskAttemptId: TaskAttemptId): Seq[BlockId] = {
    val blocksWithReleasedLocks = mutable.ArrayBuffer[BlockId]()

    val readLocks = synchronized {
      readLocksByTask.remove(taskAttemptId).getOrElse(ImmutableMultiset.of[BlockId]())
    }
    val writeLocks = synchronized {
      writeLocksByTask.remove(taskAttemptId).getOrElse(Seq.empty)
    }

    for (blockId <- writeLocks) {
      infos.get(blockId).foreach { info =>
        assert(info.writerTask == taskAttemptId)
        info.writerTask = BlockInfo.NO_WRITER
      }
      blocksWithReleasedLocks += blockId
    }
    readLocks.entrySet().iterator().asScala.foreach { entry =>
      val blockId = entry.getElement
      val lockCount = entry.getCount
      blocksWithReleasedLocks += blockId
      synchronized {
        get(blockId).foreach { info =>
          info.readerCount -= lockCount
          assert(info.readerCount >= 0)
        }
      }
    }

    synchronized {
      notifyAll()
    }
    blocksWithReleasedLocks
  }

  /** Returns the number of locks held by the given task.  Used only for testing. */
  private[storage] def getTaskLockCount(taskAttemptId: TaskAttemptId): Int = {
    readLocksByTask.get(taskAttemptId).map(_.size()).getOrElse(0) +
      writeLocksByTask.get(taskAttemptId).map(_.size).getOrElse(0)
  }

  /**
    * Returns the number of blocks tracked.
    */
  def size: Int = synchronized {
    infos.size
  }

  /**
    * Return the number of map entries in this pin counter's internal data structures.
    * This is used in unit tests in order to detect memory leaks.
    */
  private[storage] def getNumberOfMapEntries: Long = synchronized {
    size +
      readLocksByTask.size +
      readLocksByTask.map(_._2.size()).sum +
      writeLocksByTask.size +
      writeLocksByTask.map(_._2.size).sum
  }

  /**
    * Returns an iterator over a snapshot of all blocks' metadata. Note that the individual entries
    * in this iterator are mutable and thus may reflect blocks that are deleted while the iterator
    * is being traversed.
    */
  def entries: Iterator[(BlockId, BlockInfo)] = synchronized {
    infos.toArray.toIterator
  }
```

### removeBlock 和 clear

这两个方法严格来说，不应该放在一起，偷个懒啦。
removeBlock 只是用来删除给定的 block 并释放写锁。
clear 则用于清空所有的 block 读锁、写锁信息。
```scala
/**
    * Removes the given block and releases the write lock on it.
    * 
    * This can only be called while holding a write lock on the given block.
    */
  def removeBlock(blockId: BlockId): Unit = synchronized {
    logTrace(s"Task $currentTaskAttemptId trying to remove block $blockId")
    infos.get(blockId) match {
      case Some(blockInfo) =>
        if (blockInfo.writerTask != currentTaskAttemptId) {
          throw new IllegalStateException(
            s"Task $currentTaskAttemptId called remove() on block $blockId without a write lock")
        } else {
          infos.remove(blockId)
          blockInfo.readerCount = 0
          blockInfo.writerTask = BlockInfo.NO_WRITER
          writeLocksByTask.removeBinding(currentTaskAttemptId, blockId)
        }
      case None =>
        throw new IllegalArgumentException(
          s"Task $currentTaskAttemptId called remove() on non-existent block $blockId")
    }
    notifyAll()
  }

  /**
    * Delete all state. Called during shutdown.
    */
  def clear(): Unit = synchronized {
    infos.valuesIterator.foreach { blockInfo =>
      blockInfo.readerCount = 0
      blockInfo.writerTask = BlockInfo.NO_WRITER
    }
    infos.clear()
    readLocksByTask.clear()
    writeLocksByTask.clear()
    notifyAll()
  }
```
