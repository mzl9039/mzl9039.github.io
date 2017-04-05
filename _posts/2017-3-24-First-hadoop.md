---
layout:     post
title:      First Hadoop
subtitle:   Hadoop 测试
date:       2017-03-23 11:12:13
author:     mzl
catagory:   true
tags:
    - hadoop
    - learn
    - first
    big data
---
# 接触 Hadoop
## 安装 Hadoop
### 安装相关环境
1. 根据 [Hadoop 官网](http://hadoop.apache.org/) 要求，hadoop 安装需要 GNU/Linux 平台（Windows平台也支持，但步骤不在官网上）,
2. 新版本的 Hadoop 需要 JDK8
3. shh 和 pdsh 需要安装
4. 下载 Hadoop 并且解压安装
5. 设置 Hadoop 目录下 etc/hadoop/hadoop-env.sh 里的 JAVA_HOME 变量
6. 测试命令 bin/hadoop ，若出现 hadoop 的使用文档，则 hadoop 初步安装完成。
### 启动 Hadoop
Hadoop 有三种模式：
* Local Mode (standalone)
* Pseudo-Distributed Mode
* Fully-Distributed Mode
Hadoop 被默认配置为以 **非并行** 的 Mode 运行，即以 Local Mode 运行。
#### 三种模式运行
##### Local Mode(Standalone)
这种模式下，Hadoop 作为一个单独的 Java 进程运行，这样全球调试，测试代码为：
``` shell
cd $HADOOP_HOME
mkdir input
cp etc/hadoop/*.xml
bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.0.0-alpha2.jar grep input output 'dfs[a-z.]+'
cat output/*
```
##### Pseudo-Distributed Mode
Hadoop 在 Pseudo-Dictionary 模式下，可以在单节点上运行，这里每个 hadoop deamon 都是一个分开的 java process
首先使用如下配置：
etc/hadoop/core.site.xml:
``` xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost:9000</value>
    </property>
</configuration>
```
etc/hadoop/hdfs-site.xml
``` xml

<configuration>
    <property>
        <name>dfs.replication</name>
        <value>hdfs://localhost:9000</value>
    </property>
</configuration>
```
检查能否以 ssh 的方式连接：
ssh localhost
因为个人以前配置过，所以不会有问题。
在 **本地** 运行 MapReduce 的 Job，下面是简单的步骤：
1. 格式化 namenode: ```bin/hdfs namenode -format``
2. 启动 namenode 镜像和 datanode 镜像：```sbin/start-dfs.sh``
    这里 hadoop 镜像的日志会写个 $HADOOP_LOG_DIR 目录，默认是 $HADOOP_HOME/logs
3. 检查 Web 接口，默认地址为：```http://localhost:9870/``
4. 使 HDFS 目录能执行 MapReduce Job:
```
bin/hdfs dfs -mkdir /user
bin/hdfs dfs -mkdir /user/<username>
```
5. 把输入的文件拷贝到分布式文件系统中：
```
bin/hdfs dfs -mkdir input
bin/hdfs dfs -put etc/hadoop/*.xml input
```
6. 运行程序：
```
bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.0.0-alpha2.jar grep input output 'dfs[a-z.]+'
```
7. 将输出文件从分布式文件系统拷贝到本地文件系统并检查：
```
bin/hdfs dfs -get output output
cat output/*
```
8. 结束镜像运行：
```
sbin/stop-dfs.sh
```

在Pseudo Distributed 模式下，在 **Yarn** 上运行 MapReduce Job：

1. 配置如下：
etc/hadoop/mapred-site.xml
```
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
</configuration>
```
etc/hadoop/yarn-site.xml
```
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce-shuffle</value>
    </property>
    <property>
        <name>yarn.nodemanager.env-whitelist</name>
        <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_MAPRED_HOME</value>
    </property>
</configuration>
```
