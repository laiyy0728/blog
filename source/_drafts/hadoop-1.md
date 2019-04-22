---
title: Hadoop 学习（1） 基础概念
categories: hadoop
date: 2019-04-18 23:19:13
updated: 2019-04-18 23:19:13
tags:
---


# Hadoop

Hadoop 是 Apache 基金会所开发的分布式系统基础架构，主要解决海量数据的存储、分析计算问题。Hadoop 通常是指一个更广泛的概念：Hadoop 生态圈（包括 HBase、Spark 等）

<!-- more -->

Hadoop 包含的模块，以及 1.X、2.X 的区别：
![Hadoop](/images/hadoop/hadoop.png)

---

## HDFS

HDFS 架构：
- NameNode：存储文件的元数据，如：文件名、目录结构、文件属性（生成时间、副本数、文件权限），以及每个文件的快列表和快所在的 DataNode
- DateNode：在本地文件系统存储文件块数据，以及块数据的校验和
- Secondary NameNode：用来监控 HDFS 状态的辅助后台程序，每个一段时间获取 HDFS 元数据快照。


## Yarn

- ResourceManager：
> 处理客户端请求
> 监控 NodeManager
> 启动、监控 ApplicationMaster
> 资源的分配、调度
- NodeManager
> 管理单个节点上的资源
> 处理来自 ResourceManager 的命令
> 处理 ApplicationMaster 的命令
- ApplicationMaster
> 负责数据切分
> 为应用程序申请资源并分配给内部任务
> 任务的监控和容错
- Container：YARN 中资源的抽象，封装了节点上多维度资源，如：内存、CPU、磁盘、网络等，服务 ApplicationMaster

![yarn](/images/hadoop/yarn.png)

## MapReduce

MapReduce 将计算过程分为两个阶段：Map 阶段、Reduce 阶段

- Map 阶段：并行处理输入数据
- Reduce 阶段：对 Map 结果进行汇总

![MapReduce](/images/hadoop/map-reduce.png)


## 大数据生态体系
![大数据生态体系](/images/hadoop/life-cycle.png)

---

# Hadoop 环境搭建