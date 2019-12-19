---
title: zookeeper-1
date: 2019-12-19 13:55:16
updated: 2019-12-19 13:55:16
categories:
    [zookeeper]
tags:
    [zookeeper]
---

Zookeeper 是一个开源的分布式的，为分布式应用提供协调服务器的项目。

<!-- more -->

# Zookeeper

Zookeeper 从设计模式角度来理解，是一个基于 `观察者模式` 设计的分布式服务管理框架，它负责存储和管理数据，然后接受 `观察者` 的注册，一旦数据的状态发生变化，Zookeeper 就会负责 `通知以及在 Zookeeper 上注册的观察者`，让其作出相应的反应。

Zookeeper 可以解释为 `文件系统` + `通知机制` 的整合。

## 特点

> Zookeeper 是一个 `领导者(Leader)`、多个 `跟随者(Follower)` 组成的集群

> 集群中只要有半数以上的节点存活，Zookeeper 集群就能正常服务。(半数=以上是指，如果有 5 台服务器，则最少有 3 台正常；如果有 6 台服务器，最少有 4 台正常)

> 全局数据一致：每个 Server 上保存一份相同的数据备份，Client 无论连接到哪台 Server，数据都是一致的

> 更新请求顺序进行：来自同一个 Client 的更新请求将按照其发送顺序依次执行

> 数据更新的原子性：一次数据更新，要么成功，要么失败

> 实时性：在一定时间范围内，Client 能读取到最新数据


## 数据结构

Zookeeper 的数据模型结构与 Unix 文件系统类似，整体上可以看做是一棵树，每个节点称作是一个 `ZNode`。每个 ZNode 默认存储 `1MB` 数据，每个 ZNode 都可以通过其 `路径唯一标识`。

![Zookeeper 数据结构](/images/zk/zk.png)

## 应用场景

Zookeeper 提供的服务包括：`统一命名管理`、`统一配置管理`、`统一集群管理`、`服务器节点动态上下线`、`软负载均衡`等

> 统一命名管理

在分布式环境下，经常需要对 `应用/服务` 进行统一命名，便于识别。类似于：ip 不容易记，而域名容易记。

> 统一配置管理

1. 在分布式环境下，配置文件同步非常重要

一般要求一个集群中，所有节点的配置信息是一一致的；
对配置文件修改后，往往希望能够快速同步到各个节点上

2. 配置管理可以交给 Zookeeper 实现

可以将配置信息写入 Zookeeper 上的一个 ZNode；
各个客户端服务器监听这个 ZNode；
一旦 ZNode 中数据被修改，Zookeeper 将通知各个客户端服务器

> 统一集群管理

1. 分布式环境下下，实时掌握各个节点的状态是非常必要的

可根据节点实时状态做一些调整

2. Zookeeper 可以实现实时监控节点状态变化

可以将节点信息写入 Zookeeper 上的一个 ZNode；
监听这个 ZNode 获取它的实时状态变化

> 服务器节点动态上下线

客户端能实时洞察到服务器上下线变化

> 软负载均衡

在 Zookeeper 中记录每台服务器的访问数，让访问数最少的服务器去处理最新的客户端请求


---

# 安装

以 [Zookeeper 3.4.10](https://archive.apache.org/dist/zookeeper/zookeeper-3.4.10/) 为例。

## 本地模式安装

***省略 JDK 安装步骤***

> 将 Zookeeper 的压缩包拷贝到服务器上，解压到 /opt/software

```
tar -zxf zookeeper-3.4.10.tar.gz -C /opt/module/
```

> 修改配置

1. 将 `/opt/module/zookeeper-3.4.10/conf/` 下的 `zoo_sample.cfg` 复制一份为 `zoo.cfg`

2. 修改 dataDir 路径

```conf
[root@hadoop02 conf]# vim zoo.cfg 
# 心跳，2秒一次
tickTime=2000
# Leader 和 Follower 之间刚启动时进行通信的最大延迟时间：10 个心跳。如果超过 10 个心跳都没有通信，则认为 Leader 和 Follower 连接失败
initLimit=10
# 同步频率，5 个心跳；启动成后，5 个心跳没有通信，认为 Leader、Follower 连接失败
syncLimit=5
# 数据存储位置
dataDir=/opt/module/zookeeper-3.4.10/zkData
# 客户端的端口号，即 Client 访问 Zookeeper 时使用哪个端口通信
clientPort=2181
```

3. 在 `ZOOKEEPER_HOME` 中创建 zkData 文件夹

> 启动 Zookeeper 服务、查看启动结果

```sh
# 启动服务
[root@hadoop02 zookeeper-3.4.10]# bin/zkServer.sh start
ZooKeeper JMX enabled by default
Using config: /opt/module/zookeeper-3.4.10/bin/../conf/zoo.cfg
Starting zookeeper ... STARTED

# 查看启动结果
[root@hadoop02 zookeeper-3.4.10]# bin/zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /opt/module/zookeeper-3.4.10/bin/../conf/zoo.cfg
Mode: standalone

# 查看进程
[root@hadoop02 zookeeper-3.4.10]# jps
1425 Jps
1369 QuorumPeerMain
```

> 启动客户端

```
[root@hadoop02 zookeeper-3.4.10]# bin/zkCli.sh 
Connecting to localhost:2181
2019-12-19 14:48:02,837 [myid:] - INFO  [main:Environment@100] - Client environment:zookeeper.version=3.4.10-39d3a4f269333c922ed3db283be479f9deacaa0f, built on 03/23/2017 10:13 GMT
2019-12-19 14:48:02,839 [myid:] - INFO  [main:Environment@100] - Client environment:host.name=hadoop02
2019-12-19 14:48:02,839 [myid:] - INFO  [main:Environment@100] - Client environment:java.version=1.8.0_144
2019-12-19 14:48:02,841 [myid:] - INFO  [main:Environment@100] - Client environment:java.vendor=Oracle Corporation
2019-12-19 14:48:02,841 [myid:] - INFO  [main:Environment@100] - Client environment:java.home=/opt/module/jdk1.8.0_144/jre
2019-12-19 14:48:02,841 [myid:] - INFO  [main:Environment@100] - Client environment:java.class.path=/opt/module/zookeeper-3.4.10/bin/../build/classes:/opt/module/zookeeper-3.4.10/bin/../build/lib/*.jar:/opt/module/zookeeper-3.4.10/bin/../lib/slf4j-log4j12-1.6.1.jar:/opt/module/zookeeper-3.4.10/bin/../lib/slf4j-api-1.6.1.jar:/opt/module/zookeeper-3.4.10/bin/../lib/netty-3.10.5.Final.jar:/opt/module/zookeeper-3.4.10/bin/../lib/log4j-1.2.16.jar:/opt/module/zookeeper-3.4.10/bin/../lib/jline-0.9.94.jar:/opt/module/zookeeper-3.4.10/bin/../zookeeper-3.4.10.jar:/opt/module/zookeeper-3.4.10/bin/../src/java/lib/*.jar:/opt/module/zookeeper-3.4.10/bin/../conf:
2019-12-19 14:48:02,841 [myid:] - INFO  [main:Environment@100] - Client environment:java.library.path=/opt/module/protobuf-2.5.0:/usr/java/packages/lib/amd64:/usr/lib64:/lib64:/lib:/usr/lib
2019-12-19 14:48:02,841 [myid:] - INFO  [main:Environment@100] - Client environment:java.io.tmpdir=/tmp
2019-12-19 14:48:02,841 [myid:] - INFO  [main:Environment@100] - Client environment:java.compiler=<NA>
2019-12-19 14:48:02,841 [myid:] - INFO  [main:Environment@100] - Client environment:os.name=Linux
2019-12-19 14:48:02,841 [myid:] - INFO  [main:Environment@100] - Client environment:os.arch=amd64
2019-12-19 14:48:02,842 [myid:] - INFO  [main:Environment@100] - Client environment:os.version=3.10.0-693.el7.x86_64
2019-12-19 14:48:02,842 [myid:] - INFO  [main:Environment@100] - Client environment:user.name=root
2019-12-19 14:48:02,842 [myid:] - INFO  [main:Environment@100] - Client environment:user.home=/root
2019-12-19 14:48:02,842 [myid:] - INFO  [main:Environment@100] - Client environment:user.dir=/opt/module/zookeeper-3.4.10
2019-12-19 14:48:02,843 [myid:] - INFO  [main:ZooKeeper@438] - Initiating client connection, connectString=localhost:2181 sessionTimeout=30000 watcher=org.apache.zookeeper.ZooKeeperMain$MyWatcher@5c29bfd
Welcome to ZooKeeper!
2019-12-19 14:48:02,866 [myid:] - INFO  [main-SendThread(localhost:2181):ClientCnxn$SendThread@1032] - Opening socket connection to server localhost/127.0.0.1:2181. Will not attempt to authenticate using SASL (unknown error)
JLine support is enabled
2019-12-19 14:48:02,937 [myid:] - INFO  [main-SendThread(localhost:2181):ClientCnxn$SendThread@876] - Socket connection established to localhost/127.0.0.1:2181, initiating session
2019-12-19 14:48:02,955 [myid:] - INFO  [main-SendThread(localhost:2181):ClientCnxn$SendThread@1299] - Session establishment complete on server localhost/127.0.0.1:2181, sessionid = 0x16f1ce7c5fd0000, negotiated timeout = 30000

WATCHER::

WatchedEvent state:SyncConnected type:None path:null
[zk: localhost:2181(CONNECTED) 0] 
```

> 退出客户端 

```
[zk: localhost:2181(CONNECTED) 5] quit
Quitting...
2019-12-19 14:49:22,721 [myid:] - INFO  [main:ZooKeeper@684] - Session: 0x16f1ce7c5fd0000 closed
2019-12-19 14:49:22,722 [myid:] - INFO  [main-EventThread:ClientCnxn$EventThread@519] - EventThread shut down for session: 0x16f1ce7c5fd0000
```

> 关闭服务

```
[root@hadoop02 zookeeper-3.4.10]# bin/zkServer.sh stop
ZooKeeper JMX enabled by default
Using config: /opt/module/zookeeper-3.4.10/bin/../conf/zoo.cfg
Stopping zookeeper ... STOPPED
```

## 分布式安装

在 hadoop 基础上，在 hadoop02、hadoop03、hadoop04 三个节点上部署 Zookeeper

> 在 hadoop02 上创建 zkData 文件夹，并在文件夹内创建一个 `myid` 文件
> 将 zookeeper 文件夹分发到剩余两台机器上

```
xsync /opt/module/zookeeper-3.4.10/
```

> 修改三台服务器上的 `myid` 文件，文件内容分别为 2、3、4
> 修改 hadoop02 上的 zoo.cfg 文件，并分发到其他服务器上

```conf
####################cluster####################
# 注意，必须以 server. 开头， . 后面的数字是 myid 文件中配置的数字
# hadoop02 是服务器的 ip 地址或 hostname
# 2888：服务器与集群中的 Leader 交换信息的端口
# 3888：如果 Leader 挂了，需要一个端口来重新进行选举 
server.2=hadoop02:2888:3888
server.3=hadoop03:2888:3888
server.4=hadoop04:2888:3888
```


> 分别启动 Zookeeper，查看状态

```sh
# hadoop02
[root@hadoop02 zookeeper-3.4.10]# bin/zkServer.sh start
ZooKeeper JMX enabled by default
Using config: /opt/module/zookeeper-3.4.10/bin/../conf/zoo.cfg
Starting zookeeper ... STARTED

[root@hadoop02 zookeeper-3.4.10]# bin/zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /opt/module/zookeeper-3.4.10/bin/../conf/zoo.cfg
Mode: follower


# hadoop03
[root@hadoop03 zookeeper-3.4.10]# bin/zkServer.sh start
ZooKeeper JMX enabled by default
Using config: /opt/module/zookeeper-3.4.10/bin/../conf/zoo.cfg
Starting zookeeper ... STARTED

[root@hadoop03 zookeeper-3.4.10]# bin/zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /opt/module/zookeeper-3.4.10/bin/../conf/zoo.cfg
Mode: leader


# hadoop04
[root@hadoop04 zookeeper-3.4.10]# bin/zkServer.sh start
ZooKeeper JMX enabled by default
Using config: /opt/module/zookeeper-3.4.10/bin/../conf/zoo.cfg
Starting zookeeper ... STARTED

[root@hadoop04 zookeeper-3.4.10]# bin/zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /opt/module/zookeeper-3.4.10/bin/../conf/zoo.cfg
Mode: follower
```

---


# 客户端命令行操作

| 命令基本语法 | 描述 |
| :-: | :-: |
| help | 显示所有操作命令 |
| ls [path] | 


# Zookeeper 内部原理

## 选举机制

> 半数机制：集群中半数以上的机器存活，集群可用。所以 Zookeeper 适合安装 ***奇数台服务器***

> Zookeeper 虽然在配置文件中没有指定 Master 和 Slave，但是 Zookeeper 工作室，是有一个节点为 Leader，其他为 Follower；Leader 是通过 `内部选举机制临时产生`

### 选举过程

假设有舞台服务器组成 Zookeeper 集群，它们的 id 为 `1~5`，同时，它们都是最新启动的，没有历史数据，所有配置一样。假设服务器依次启动，其选举过程为：

1. 服务器 A 启动，此时只有一台服务器启动，它发出的报文没有任何响应，选举状态为 `LOOKING`
2. 服务器 B 启动，与 A 通信，互相交换自己的选举结果。由于两者都没有历史数据，所以 id 较大的 B 胜出；但是，由于没有超过半数以上的服务器同意（5台服务器，半数以上是3)，所以 A、B 继续保持 `LOOKING`
3. 服务器 C 启动，与 A、B 通信，根据前面的分析，此时 C 是集群中的 Leader。
4. 服务器 D 启动，与 A、B、C 通信，理论上来说应该是 D 为 Leader，但是 C 已经成为了 Leader， 所以 D 只能成为 Follower
5. 服务器 E 启动，同样与 D 一样，只能当 Follower


## 节点类型

> 持久型

客户端和服务器端断开链接后，创建的节点不删除；
Zookeeper 给该节点名称进行顺序编号

创建 ZNode 时，设置顺序标识，ZNode 名称后会附加一个值，顺序号是一个单调递增的计数器，由父节点维护

***注意***：在分布式系统中，顺序号可以被用于为所有的事件进行全局排序，这样客户端可以通过顺序号推断事件的顺序

> 短暂型
 
客户端与服务器端断开链接后，创建的节点自己删除；
Zookeeper 给该节点名称进行顺序编码

