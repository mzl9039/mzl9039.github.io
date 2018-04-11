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

<div class="mermaid">
graph TD
    subgraph Driver
        MapOutputTrackerMaster-->MapOutputTrackerMasterEndpoint
    end
    subgraph Executor0
        MapOutputTrackerWorker0[MapOutputTrackerWorker]-->MapOutputTrackerMasterEndpoint
        subgraph threadPool
            ShuffleMapTask1[ShuffleMapTask]
            ShuffleMapTask2[ShuffleMapTask]
            ResultTask[0][ResultTask]
        end
    end
    subgraph Executor1
        MapOutputTrackerWorker1[MapOutputTrackerWorker]-->MapOutputTrackerMasterEndpoint
        subgraph threadPool
            ResultTask1[ResultTask]
        end
    end
</div>

# 引用：
1. [spark学习-35-Spark的Map任务输出跟踪器MapOutputTracker](https://blog.csdn.net/qq_21383435/article/details/78603123)
