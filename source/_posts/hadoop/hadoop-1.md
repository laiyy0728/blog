---
title: Hadoop 学习（1）  <br /> 基础概念、基础环境搭建安装
categories: hadoop
date: 2019-09-17 23:19:13
updated: 2019-09-17 23:19:13
tags: [hadoop, linux]
---


# Hadoop

Hadoop 是 Apache 基金会所开发的分布式系统基础架构，主要解决海量数据的存储、分析计算问题。Hadoop 通常是指一个更广泛的概念：Hadoop 生态圈（包括 HBase、Spark 等）

<!-- more -->


## Hadoop 的三大发型版本

> Apache

最原始版本，对于入门学习最好

> Cloudera

收费，在大型互联网公司用的比较多

> Hortonworks

文档比较好


## Hadoop 的优势

> 高可靠性

Hadoop 底层维护多个数据副本（默认3个），所以即使 Hadoop 某个计算元素或存储出现故障，也不会导致数据的丢失

> 高扩展性

在集群中分配任务数据，可方便的扩展数以千计的节点

> 高效性

在 MapReduce 思想下，Hadoop 是并行工作的，以加快任务处理速度

> 高容错性

自动将失败的任务重新分配执行

## 1.x 与 2.x 的区别

Hadoop 包含的模块，以及 1.X、2.X 的区别：
![Hadoop](/images/hadoop/hadoop.png)



---

# Hadoop 三大组件

## HDFS

HDFS 架构：
- NameNode(nn)：
相当于一本书的目录；存储文件的元数据，如：文件名、目录结构、文件属性（生成时间、副本数、文件权限），以及每个文件的快列表和快所在的 DataNode
- DateNode(dn)：
相当于目录对应的具体内容；在本地文件系统存储文件块数据，以及块数据的校验和
- Secondary NameNode(2nn)：用来监控 HDFS 状态的辅助后台程序，每个一段时间获取 HDFS 元数据快照。


## Yarn

ResourceManager（相当于老板） > NodeManager（相当于技术总监）/ApplicationMaster（相当于项目经理）

其中 NodeManager 负责某一个节点，ApplicationMaster 负责节点中的某个任务

- ResourceManager
> 处理客户端请求：管理整个服务器集群资源（磁盘、cpu等）
> 监控 NodeManager
> 启动、监控 ApplicationMaster（在集群上运行的任务）
> 资源的分配、调度

- NodeManager
> 管理单个节点上的资源
> 处理来自 ResourceManager 的命令
> 处理 ApplicationMaster 的命令

- ApplicationMaster
> 负责数据切分
> 为应用程序申请资源并分配给内部任务
> 任务的监控和容错

- Container：为 ApplicationMaster 服务
> YARN 中资源的抽象，封装了节点上多维度资源，如：内存、CPU、磁盘、网络等，服务 ApplicationMaster

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

环境准备

> CentOS 7
> JDK 1.8
> Hadoop 2.9.2


## CentOS 设置

### 修改网卡

使用 ROOT 用户，设置 NAT 模式网络连接、设置静态 IP （防止 dhcp 导致 ip 变化）
```
vim /etc/sysconfig/network-scripts/ifcfg-ens33
```

禁用掉 ipv6，将获取 ip 的方式从 `dhcp` 改为 `static`，设置静态 ip 地址 `IPADDR`、网关 `GATEWAY`、dns `DNS1`

```
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=no
IPV6_AUTOCONF=no
IPV6_DEFROUTE=no
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens33
UUID=672a42a7-bbbb-453e-8ebe-ca0ae27eef49
DEVICE=ens33
ONBOOT=yes



IPADDR=192.168.52.100
GATEWAY=192.168.52.2
DNS1=192.168.52.2
```

*** 注意：网关需要与虚拟机中的 NAT 网卡设置一致！***

![网卡设置](/images/hadoop/network-1.png)
![网卡设置](/images/hadoop/network-2.png)
![网卡设置](/images/hadoop/network-3.png)

***另外，也需要将 VM8 网卡设置一个静态ip，且这个静态 ip 不能与虚拟机的静态 ip一致，必须在一个网段***
![网卡设置](/images/hadoop/network-4.png)


### 修改 hostname、hosts

```
vim /etc/sysconfig/network

NETWORKING=yes
HOSTNAME=hadoop_01
```

```
vim /etc/hosts

192.168.52.100 hadoop_01
192.168.52.101 hadoop_02
192.168.52.102 hadoop_03
192.168.52.103 hadoop_04
```

其中，`hosts` 文件中的 `hadoop_02`、`hadoop_03`、`hadoop_04` 暂时还没有，先配置上，为后面集群做准备。

## 安装 JDK、Hadoop

寻找一个文件夹或创建一个空文件夹，作为 jdk、hadoop 的安装目录。本例以 /opt 文件夹为例。

在 /opt 文件夹下创建 `module` 文件夹，存放 jdk、hadoop 安装文件，创建 `software` 文件夹，存在 jdk、hadoop 安装包

```
cd /opt
mkdir module software
```

利用 xftp 将下载好的 jdk1.8、hadoop 2.9.2 存入 software 文件夹下，解压安装到 module 文件夹下

```
cd software

tar -zxf jdk-8u144-linux-x64.tar.gz -C ../module/
tar -zxf hadoop-2.9.2.tar.gz -C ../module/
```

设置 jdk、hadoop 环境变量

```
vim /etc/profile
```

在文件最后添加：
```
export JAVA_HOME=/opt/module/jdk1.8.0_144
export HADOOP_HOME=/opt/module/hadoop-2.9.2
export PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
```

应用环境变量：
```
source /etc/profile
```

使用 `java`、`javac`、`hadoop` 命令，测试环境变量设置是否成功
