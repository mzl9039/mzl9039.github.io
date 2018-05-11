---
layout:     post
title:      Spark MemoryManager 内存管理
subtitle:   
date:       2018-05-09 00:00:00
author:     mzl
catalog:    true
tags:
    - spark
    - memoryManager
    - task
---

{:toc}

# MemoryManager 内存管理

spark 的计算模型又叫"内存计算模型"，虽然 spark 也支持磁盘存储，但其主要的优势在于充分利用内存计算。
它本身也对内存的使用和管理进行了一系列的优化。

内存的使用和管理，是两个不同的问题。内存的管理，在于其内存模型，如何充分利用内存完成计算任务。而内存使用，
则主要在于如何申请内存并优化读写，尤其是 rdd 缓存时申请内存和 task 计算时申请内存两种方式的不同。本文重点
分析内存的管理模型，内存的使用，根据需要在本文中完成，或在以后的博文中再进行分析。

由于 spark 内存管理模型，网上分析的文章已经很多，这里会引用一些网上的文章。

## 

## 引用

1. [Apache Spark 内存管理详解](https://www.ibm.com/developerworks/cn/analytics/library/ba-cn-apache-spark-memory-management/index.html?ca=drs-&utm_source=tuicool&utm_medium=referral)
