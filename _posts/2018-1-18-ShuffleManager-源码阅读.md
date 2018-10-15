---
layout:     post
title:      "Spark ShuffleManager源码阅读"
subtitle:   ""
date:       2018-1-18 22:59:59
author:     mzl
catalog:    true
tags:
    - spark
    - ShuffleManager
---

ShuffleManager 是一个trait
```scala
private[spark] trait ShuffleManager {
    ...
}
```
