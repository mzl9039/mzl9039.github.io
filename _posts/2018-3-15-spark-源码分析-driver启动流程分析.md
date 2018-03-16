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

所以我们知道，启动 driver 是在初始化 sc 的时候完成的，这里是我们分析的起点。从以前的[博客-Spark 任务分发与执行流程](https://mzl9039.github.io/2018/03/05/spark-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-%E4%BB%BB%E5%8A%A1%E5%88%86%E5%8F%91%E4%B8%8E%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B.html)
中提到，
