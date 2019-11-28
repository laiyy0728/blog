---
title: Hadoop（7） <br/> HDFS 数据流
date: 2019-11-28 09:24:43
updated: 2019-11-28 09:24:48
categories:
    [hadoop]
tags: [hadoop]
---

此前使用 HDFS API 测试完成了一些对 HDFS 的基础操作（上传、下载、重命名、拷贝等），接下来需要了解一下 HDFS 对于数据是怎么操作的

<!-- more -->

# HDFS 写数据流程

1. 使用 FileSystem.get 创建一个`分布式文件系统`客户端，向 NameNode 请求上传文件
2. NameNode 检查 HDFS 中是否有待上传的文件（根据路径、文件名判断），如果存在该文件，则报错`文件已存在`
3. 如果 NameNode 检查后，HDFS 没有待上传的文件，则开始响应上传文件
4. 请求上传第一个 block（根据配置不同，block 大小也不同），此时会向 DataNode 请求，由 DataNode 决定可以上传到哪几个节点上
5. DataNode 返回可以上传的节点（判断条件：节点距离，负载）
6. FileSystem 创建输出流（FsDataOutputStream），与 DataNode 建立通道（串行）
7. DataNode 应答，所有可上传节点应答成功后，开始传输数据
8. 所有数据传输完成后，通知 NameNode

![hdfs 写数据流程](/images/hadoop/client/write-data.png)


## 节点距离计算

`节点距离：两个节点到最近的共同祖先的距离总和`

HDFS 写数据过程中，NameNode 会选择距离上传数据最近距离的 DataNode 接收数据，此时需要计算节点距离。

![节点距离](/images/hadoop/client/distance.png)

> (d1-r1-n0, d1-r1-n0)，由于这两个节点在同一个服务器上，此时距离为 0。即：同一节点上的进程距离为 0。
> (d1-r1-n1, d1-r1-n2)，由于这两个节点都在同一个机架上，所以 n1、n2 的共同祖先都为 r1，此时距离为 1+1=2
> (d1-r2-n0, d1-r3-n2)，这两个节点在同一个集群的不同机架上，即这两个节点的共同祖先为 d1，节点到集群还需要经过机架，所以这两个节点到共同祖先的距离都为 2，则节点距离为 2+2=4
> (d1-r2-n1, d2-r4-n1)，这两个节点也不在同一个集群，则共同祖先为最外围的“网段”，此时每个节点到“网段”的距离都为 3，所以节点距离为 3+3=6

## 机架感知（副本存储节点选择）