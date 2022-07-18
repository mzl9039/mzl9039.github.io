---
layout:     post
title:      PySpark 使用的几个问题
subtitle:
date:       2022-07-18 00:00:00
author:     mzl
catalog:    true
tags:
    - spark
    - pyspark
    - 问题记录
    - 笔记
---

{:toc}

# 问题列表

* PATH 变量不正确导致 Error: **Java gateway process exited before sending its port number**
* 公共环境没有需要的 library 报 No Module Named XXX Error

## 背景介绍

今天工作时, 调用 pyspark 计算行政区面积(使用 shapely)及相应 poi 密度.

## 问题1

估计是由于 .bash_probile 的问题, 导致在非 base 环境下, 系统默认的 PATH 环境变量消失(待排查)

### 解决

在项目的 .vscode 路径下添加 .env 文件, 文件内容为系统默认的 PATH 环境变量; 然后执行 jupyter 时先执行如下代码:

```python
import os
with open("/path_to_project/.vscode/.env", 'r') as f:
    line = f.readline()

line = line.split("=")[1]
os.environ["PATH"] = os.environ["PATH"] + ":" + line
print(os.environ['PATH'].split(':'))
```

## 问题2

pyspark 环境变量设置如下:

```python
import os

os.environ["SPARK_HOME"] = '/usr/share/spark'
os.environ["PYSPARK_PYTHON"] = '/usr/share/miniconda2/envs/py36/bin/python'
os.environ["PYSPARK_DRIVER_PYTHON"] = '/usr/share/miniconda2/envs/py36/bin/python'
```

pyspark conf 默认设置如下:

```python
SPARK_CONF = SparkConf() \
    .set('spark.locality.wait', '1000ms') \
    .set("spark.driver.maxResultSize", '30g')\
    .set("spark.driver.memory", '10g') \
    .set('spark.sql.hive.manageFilesourcePartitions', 'false') \
    .set('spark.yarn.queue', 'ds-regular') \
    .set('spark.executor.cores', '4') \
    .set('spark.executor.memory', '10g') \
    .set('spark.executor.instances', '128') \
    .set("spark.kryoserializer.buffer.max", "2047m")\
    .set("spark.speculation","false")

sc = SparkContext(appName='project_name_zhiliang.mo', conf=SPARK_CONF)
spark = SparkSession \
    .builder \
    .enableHiveSupport() \
    .getOrCreate()
```

### 解决

解决方案是: 让 pyspark 使用本地的 python 环境

具体就是:
1. 到 conda 环境下, 把当前 env 整个打为 zip 包, 并提交到 hdfs 上
2. 修改 pyspark 环境变量见下
3. 修改 pyspark conf 见下

修改后的 pyspark 环境变量设置如下:

```python
os.environ["SPARK_HOME"] = '/usr/share/spark'  # add this before "import
# 注意这里的 my_py36 和 mzl_py36: my_py36 为spark 将 python 环境解压前所在的路径名, mzl_py36 为当前 python env 环境包, 即zip包解压后的文件名
os.environ["PYSPARK_PYTHON"] = "./my_py36/mzl_py36/bin/python"
os.environ["PYSPARK_DRIVER_PYTHON"] = "/ldap_home/zhiliang.mo/.conda/envs/mzl_py36/bin/python"
```

修改后的 pyspark conf 默认设置如下:

```python
SPARK_CONF = SparkConf() \
    .set('spark.locality.wait', '1000ms') \
    .set("spark.driver.maxResultSize", '30g')\
    .set("spark.driver.memory", '10g') \
    .set('spark.sql.hive.manageFilesourcePartitions', 'false') \
    .set('spark.yarn.queue', 'ds-regular') \
    .set('spark.executor.cores', '4') \
    .set('spark.executor.memory', '10g') \
    .set('spark.executor.instances', '128') \
    .set("spark.kryoserializer.buffer.max", "2047m")\
    .set("spark.speculation","false") \
    .set('spark.sql.session.timeZone', 'GMT+7')\
    .set("spark.sql.execution.arrow.enabled", "true") \
    .set("spark.sql.crossJoin.enabled", True)  \
    .set("spark.network.timeout", '3600s') \
    .set("spark.executor.heartbeatInterval", '1600s') \
    .set("spark.sql.shuffle.partitions", '4096') \
    .set("spark.debug.maxToStringFields", '500')\
    .set("spark.blacklist.enabled", 'false')\
    .set('spark.yarn.dist.archives', 'hdfs://R2/user/zhiliang.mo/mzl_py36.zip#my_py36') # 注意这里的 my_py36 和上面的环境变量对应

sc = SparkContext(appName='project_name_zhiliang.mo', conf=SPARK_CONF)
spark = SparkSession \
    .builder \
    .enableHiveSupport() \
    .getOrCreate()
```