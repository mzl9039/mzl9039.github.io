---
layout:     post
title:      stage 划分
subtitle:   stage 划分分析
date:       2018-04-07 00:00:00
author:     mzl
catalog:    true
tags:
    - spark
    - stage
    - ResultStage
    - ShuffleMapStage
---

{:toc}

# Stage 划分

前面的[博客](https://mzl9039.github.io/2018/03/02/spark-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-%E4%BB%BB%E5%8A%A1%E6%8F%90%E4%BA%A4%E6%B5%81%E7%A8%8B.html)分析了 job 提交的流程,但 job 提交的过程中，对 stage 的划分，只是简单提到了，并没有详细分析。
从前文中，我们知道提交 spark job 是在触发了 action 算子后，触发了 sc 的 runJob 方法，这个方法调用 dagScheduler 的 runJob 方法来执行，然后调用 submitJob 方法，并在这个方法中，触发事件 JobSubmitted, 最后由
dagScheduler 的 handleJobSubmitted 方法来生成 finalStage，但由于 spark 的 stage 是图，这个 finalStage 必然记录了整个 stage 的图结构，以便出错时可以恢复，所以说这里完成了 stage 的划分。

<div class="mermaid">
graph TD
    action[action算子]-->|1.调用对象|SparkContext(sc)
    SparkContext-->|2.执行方法|runJob_sc[runJob]
    runJob_sc-->|3.调用对象|dagScheduler(dagScheduler)
    dagScheduler-->|4.执行方法|runJob_ds[runJob]
    runJob_ds-->|5.执行方法|submitJob
    submitJob-->|6.触发事件|JobSubmitted
    JobSubmitted-->|7.执行方法|handleJobSubmitted
    handleJobSubmitted-->|8.执行方法|createResultStage[生成finalStage]
</div>

本文打算通过源码对 spark job 的 stage 划分部分，做一个详细的分析。由于 job 提交的流程已经在前面的博客中分析过，这里直接从 dagScheduler 对象的方法 handleJobSubmitted 开始。

## DagScheduler 的 handleJobSubmitted 方法

这个方法在创建 finalStage 时，完成了stage 的划分，然后调用方法 submitStage 提交划分好的 stage.
所以，划分 stage 的过程，在创建 finalStage 的时候应该就完成了。

```scala
private[scheduler] def handleJobSubmitted(jobId: Int,
    finalRDD: RDD[_],
    func: (TaskContext, Iterator[_]) => _,
    partitions: Array[Int],
    callSite: CallSite,
    listener: JobListener,
    properties: Properties) {
  var finalStage: ResultStage = null
  try {
    /** New stage creation may throw an exception if, for example, jobs are run on a */
    /** HadoopRDD whose underlying HDFS files have been deleted. */
    /** 生成 finalStage, 事件上，为了记录整个 stage 图结构，finalStage 已经完成了前面所有 stage 的划分。 */
    finalStage = createResultStage(finalRDD, func, partitions, jobId, callSite)
  } catch {
    case e: Exception =>
      logWarning("Creating new stage failed due to exception - job: " + jobId, e)
      listener.jobFailed(e)
      return
  }
  
  /** 生成 ActiveJob 对象，保存 Job 信息 */
  val job = new ActiveJob(jobId, finalStage, callSite, listener, properties)
  clearCacheLocs()
  logInfo("Got job %s (%s) with %d output partitions".format(
    job.jobId, callSite.shortForm, partitions.length))
  logInfo("Final stage: " + finalStage + " (" + finalStage.name + ")")
  logInfo("Parents of final stage: " + finalStage.parents)
  logInfo("Missing parents: " + getMissingParentStages(finalStage))

  val jobSubmissionTime = clock.getTimeMillis()
  jobIdToActiveJob(jobId) = job
  activeJobs += job
  finalStage.setActiveJob(job)
  val stageIds = jobIdToStageIds(jobId).toArray
  val stageInfos = stageIds.flatMap(id => stageIdToStage.get(id).map(_.latestInfo))
  /** 这里会触发 onJobStart 方法,详见 SparkListenerBus */
  listenerBus.post(
    SparkListenerJobStart(job.jobId, jobSubmissionTime, stageInfos, properties))
  /** 这里提交已经划分完成的 stage */
  submitStage(finalStage)
}
```

## DagScheduler 的 createResultStage 方法

创建 rdd 链最后的那个 ResultStage, 由于要创建最后的 ResultStage, 必然要随之创建所有 stage，这样最后的
ResultStage 能会记录整个 stage 图结构

```scala
/** Create a ResultStage associated with the provided jobId. */
/** 参数里的 rdd 即为 rdd 链最后的那个 action rdd，用于生成 ResultStage */
private def createResultStage(
    rdd: RDD[_],
    func: (TaskContext, Iterator[_]) => _,
    partitions: Array[Int],
    jobId: Int,
    callSite: CallSite): ResultStage = {
  /** 这里根据最后的那个 rdd，生成 parent stage. 这是因为 rdd 的属性 dependencies 包含了其 dependencies 信息 */
  /** 而 dependencies 里的每个 dependency 又包含了 rdd 链上层的 rdd 信息 */
  val parents = getOrCreateParentStages(rdd, jobId)
  val id = nextStageId.getAndIncrement()
  val stage = new ResultStage(id, rdd, func, partitions, parents, jobId, callSite)
  stageIdToStage(id) = stage
  updateJobIdStageIdMaps(jobId, stage)
  stage
}
```

## DagScheduler 的 getOrCreateParentStages 方法

这个方法用于获取 rdd 的 parent stage，由于要获取 parent stage，可以知道如果 rdd 与父 rdd 之间是 NarrowDependency, 则
会继续向父 rdd 搜索，直接找到 ShuffleDependency 或 None。

```scala
/** Get or create the list of parent stages for a given RDD.  The new Stages will be created with */
/** the provided firstJobId. */
private def getOrCreateParentStages(rdd: RDD[_], firstJobId: Int): List[Stage] = {
  /** 获取当前 rdd 的 ShuffleDependency, 注意这里的重要是上一个 ShuffleDependency, 如果 rdd 链上有多个 ShuffleDependency */
  /** 不会返回整个 rdd 链上的所有 ShuffleDependency */
  getShuffleDependencies(rdd).map { shuffleDep =>
    getOrCreateShuffleMapStage(shuffleDep, firstJobId)
  }.toList
}
```

## DagScheduler 的 getShuffleDependencies 方法

```scala
/** Returns shuffle dependencies that are immediate parents of the given RDD. */
/** 返回给定 RDD 的最近的 parents 的 shuffle dependencies */
/** */
/** This function will not return more distant ancestors.  For example, if C has a shuffle */
/** dependency on B which has a shuffle dependency on A: */
/** 这个方法不会返回更远的 ancestors. 即如下图的话，只会返回 C 与 B 之间的 shuffleDep, 而不会返回 B 与 A 之间的 */
/** 但如果 C 与 B 之间是 narrowDep,则会返回 B 与 A 之间的 dep */
/** */
/** A <-(shuffle dep)- B <-(shuffle dep)- C */
/** */
/** calling this function with rdd C will only return the B <-- C dependency. */
/** */
/** This function is scheduler-visible for the purpose of unit testing. */
private[scheduler] def getShuffleDependencies(
    rdd: RDD[_]): HashSet[ShuffleDependency[_, _, _]] = {
  val parents = new HashSet[ShuffleDependency[_, _, _]]
  val visited = new HashSet[RDD[_]]
  val waitingForVisit = new Stack[RDD[_]]
  waitingForVisit.push(rdd)
  while (waitingForVisit.nonEmpty) {
    val toVisit = waitingForVisit.pop()
    if (!visited(toVisit)) {
      visited += toVisit
      toVisit.dependencies.foreach {
        case shuffleDep: ShuffleDependency[_, _, _] =>
          parents += shuffleDep
        case dependency =>
          waitingForVisit.push(dependency.rdd)
      }
    }
  }
  parents
}
```

## DagScheduler 的 getOrCreateShuffleMapStage 方法

这个方法用于创建或获取 ShuffleMapStage. 若 shuffleId 已存在，则返回；否则创建 ShuffleMapStage 并返回。

```scala
/** Gets a shuffle map stage if one exists in shuffleIdToMapStage. Otherwise, if the */
/** shuffle map stage doesn't already exist, this method will create the shuffle map stage in */
/** addition to any missing ancestor shuffle map stages. */
private def getOrCreateShuffleMapStage(
    shuffleDep: ShuffleDependency[_, _, _],
    firstJobId: Int): ShuffleMapStage = {
  /** shuffleIdToMapStage 是一个 HashMap, 若 shuffleId 已存在，则直接返回 shuffleId 对应的 mapStage */
  /** 若不存在，则创建, 注意这里创建时，需要先调用方法 getMissingAncestorShuffleDependencies 创建父 stage. */
  /** 这个创建过程，又会调用 getOrCreateParentStages, 因为就递归地完成了整个 rdd 链的遍历，创建了整个 stage 图结构 */
  shuffleIdToMapStage.get(shuffleDep.shuffleId) match {
    case Some(stage) =>
      stage

    case None =>
      /** Create stages for all missing ancestor shuffle dependencies. */
      /** 若当前 shuffleId 未保存，则先获取 shuffleDep 父 dep 并创建 ShuffleMapStage, 最后才创建当前的 shuffleDep 的 stage */
      /** 这里会根据 shuffleDep.rdd 获取其父 shuffleDep, 并通过 foreach 创建 ShuffleMapStage */
      getMissingAncestorShuffleDependencies(shuffleDep.rdd).foreach { dep =>
        /** Even though getMissingAncestorShuffleDependencies only returns shuffle dependencies */
        /** that were not already in shuffleIdToMapStage, it's possible that by the time we */
        /** get to a particular dependency in the foreach loop, it's been added to */
        /** shuffleIdToMapStage by the stage creation process for an earlier dependency. See */
        /** SPARK-13902 for more information. */
        /** 这里大致讲清楚了，SPARK-13902 是解决生成了多个相同的 ShuffleMapStage 问题的 */
        /** 虽然 getMissingAncestorShuffleDependencies 方法返回的是不在 shuffleIdToMapStage 中保存的 stage，但程序运行到 foreach 时， */
        /** 可能在这些不在 shuffleIdToMapStage 中的 stage, 已经在其它更早的 stage 创建过程中被生成并添加到 shuffleIdToMapStage */
        if (!shuffleIdToMapStage.contains(dep.shuffleId)) {
          createShuffleMapStage(dep, firstJobId)
        }
      }
      /** Finally, create a stage for the given shuffle dependency. */
      createShuffleMapStage(shuffleDep, firstJobId)
  }
}
```

## DagScheduler 的 getMissingAncestorShuffleDependencies 方法

这个方法获取最靠近当前 rdd 的 shuffleDep(不在 shuffleIdToMapStage 中), 并返回。

```scala
/** Find ancestor shuffle dependencies that are not registered in shuffleToMapStage yet */
private def getMissingAncestorShuffleDependencies(
    rdd: RDD[_]): Stack[ShuffleDependency[_, _, _]] = {
  val ancestors = new Stack[ShuffleDependency[_, _, _]]
  val visited = new HashSet[RDD[_]]
  /** We are manually maintaining a stack here to prevent StackOverflowError */
  /** caused by recursively visiting */
  val waitingForVisit = new Stack[RDD[_]]
  waitingForVisit.push(rdd)
  while (waitingForVisit.nonEmpty) {
    val toVisit = waitingForVisit.pop()
    if (!visited(toVisit)) {
      visited += toVisit
      /** getShuffleDependencies 方法前面讲过了，只返回最靠近当前 rdd 的那个 ShuffleDependency, 更早的不返回 */
      getShuffleDependencies(toVisit).foreach { shuffleDep =>
        /** 这里只处理不在 shuffleIdToMapStage 中的 stage，这里的处理，并不能解决 SPARK-13902 */
        if (!shuffleIdToMapStage.contains(shuffleDep.shuffleId)) {
          ancestors.push(shuffleDep)
          waitingForVisit.push(shuffleDep.rdd)
        } /** Otherwise, the dependency and its ancestors have already been registered. */
      }
    }
  }
  ancestors
}
```

## DagScheduler 的 createShuffleMapStage 方法

在这个方法中，会调用 getOrCreateParentStages, 这里就是在递归调用这个方法了，通过这种方式，就实现了
遍历整个 rdd 链来创建完成的 stage 图的过程。

```scala
/** Creates a ShuffleMapStage that generates the given shuffle dependency's partitions. If a */
/** previously run stage generated the same shuffle data, this function will copy the output */
/** locations that are still available from the previous shuffle to avoid unnecessarily */
/** regenerating data. */
def createShuffleMapStage(shuffleDep: ShuffleDependency[_, _, _], jobId: Int): ShuffleMapStage = {
  val rdd = shuffleDep.rdd
  val numTasks = rdd.partitions.length
  /** 这里完成了递归调用，忽略这个递归的过程，我们暂时关注当前方法的功能 */
  val parents = getOrCreateParentStages(rdd, jobId)
  val id = nextStageId.getAndIncrement()
  val stage = new ShuffleMapStage(id, rdd, numTasks, parents, jobId, rdd.creationSite, shuffleDep)

  stageIdToStage(id) = stage
  /** 这里将 stage 保存到 shuffleIdToMapStage 中 */
  shuffleIdToMapStage(shuffleDep.shuffleId) = stage
  updateJobIdStageIdMaps(jobId, stage)

  if (mapOutputTracker.containsShuffle(shuffleDep.shuffleId)) {
    /** A previously run stage generated partitions for this shuffle, so for each output */
    /** that's still available, copy information about that output location to the new stage */
    /** (so we don't unnecessarily re-compute that data). */
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
    logInfo("Registering RDD " + rdd.id + " (" + rdd.getCreationSite + ")")
    mapOutputTracker.registerShuffle(shuffleDep.shuffleId, rdd.partitions.length)
  }
  stage
}
```

# 写在最后的话

注意到上面这些方法，好几个是 getOrCreate, 说明只要第一次调用到，就可以完成整个 stage 的构建，而不需要
做 get 和 create 的区分。这样的话，在 handleJobSubmitted 方法中，先调用 getOrCreateShuffleMapStage, 而后面
submitStage 方法里的 getMissingParentStages，只需要 get 即可，虽然这个方法调用的是 getOrCreateShuffleMapStage
