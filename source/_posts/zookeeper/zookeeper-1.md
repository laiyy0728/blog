---
title: Zookeeper（一） <BR/> 安装、命令行操作
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

## help

可以显示所有操作命令

```
[zk: localhost:2181(CONNECTED) 3] help
ZooKeeper -server host:port cmd args
	stat path [watch]
	set path data [version]
	ls path [watch]
	delquota [-n|-b] path
	ls2 path [watch]
	setAcl path acl
	setquota -n|-b val path
	history 
	redo cmdno
	printwatches on|off
	delete path [version]
	sync path
	listquota path
	rmr path
	get path [watch]
	create [-s] [-e] path data acl
	addauth scheme auth
	quit 
	getAcl path
	close 
	connect host:port
```

## ls path [watch]

查看当前 ZNode 中所包含的内容；除根节点外，都不能以 / 结尾

```
[zk: localhost:2181(CONNECTED) 1] ls /
[zookeeper]
```

> 节点监听

在 hadoop03、hadoop04 上，监听 `/laiyy` 节点的变化。

```
[zk: localhost:2181(CONNECTED) 0] ls /laiyy watch
[gender0000000002, name, gender0000000001, gender0000000004, gender0000000003]
```

在 hadoop02 上修改/创建一个 `/laiyy` 节点或子节点，查看 hadoop03、hadoop04 的变化

```
[zk: localhost:2181(CONNECTED) 16] create /laiyy/email 'laiyy0728@gmail.com'
Created /laiyy/email
```

```
[zk: localhost:2181(CONNECTED) 1] 
WATCHER::

WatchedEvent state:SyncConnected type:NodeChildrenChanged path:/laiyy

```

***注意：监听方式只有一次有效，即监听到一次变化后，后续的就不再监听了***

## ls2 path [watch]

获取 ZNode 总所包含的内容，包括更新时间、次数等；除根节点外，都不能以 / 结尾

```
[zk: localhost:2181(CONNECTED) 2] ls2 /
[zookeeper]
cZxid = 0x0
ctime = Thu Jan 01 08:00:00 CST 1970
mZxid = 0x0
mtime = Thu Jan 01 08:00:00 CST 1970
pZxid = 0x0
cversion = -1
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 0
numChildren = 1
```

## create

普通创建。  -s 含有序列， -e 临时（重启或超时即消失）

***注意***： 创建节点的同时，需要写入数据，否则创建不成功

```
[zk: localhost:2181(CONNECTED) 4] create /laiyy "test_create"
Created /laiyy
[zk: localhost:2181(CONNECTED) 5] ls /
[laiyy, zookeeper]
```

> 短暂节点

```
[zk: localhost:2181(CONNECTED) 0] create -e /laiyy/name "laiyy"
Created /laiyy/name
[zk: localhost:2181(CONNECTED) 1] ls /laiyy
[name]
```

创建成功后，退出客户端重新连接查看 `/laiyy` 节点

```
[zk: localhost:2181(CONNECTED) 2] quit
Quitting...
2019-12-20 10:24:15,052 [myid:] - INFO  [main:ZooKeeper@684] - Session: 0x26f2112be2b0001 closed
2019-12-20 10:24:15,054 [myid:] - INFO  [main-EventThread:ClientCnxn$EventThread@519] - EventThread shut down for session: 0x26f2112be2b0001


[root@hadoop02 zookeeper-3.4.10]# bin/zkCli.sh 
Connecting to localhost:2181

[zk: localhost:2181(CONNECTED) 0] ls /laiyy
[]
```

> 带序号的节点（持久型）

```
[zk: localhost:2181(CONNECTED) 1] create -s /laiyy/gender 'man'
Created /laiyy/gender0000000001

[zk: localhost:2181(CONNECTED) 4] ls /laiyy
[gender0000000001]

[zk: localhost:2181(CONNECTED) 1] create -s /laiyy/gender 'man'
Created /laiyy/gender0000000002
[zk: localhost:2181(CONNECTED) 1] create -s /laiyy/gender 'man'
Created /laiyy/gender0000000003
[zk: localhost:2181(CONNECTED) 1] create -s /laiyy/gender 'man'
Created /laiyy/gender0000000004

[zk: localhost:2181(CONNECTED) 8] ls /laiyy
[gender0000000002, gender0000000001, gender0000000004, gender0000000003]
```

## get path [watch]

获取节点的值

```
[zk: localhost:2181(CONNECTED) 6] get /laiyy
test_create
cZxid = 0x200000002
ctime = Fri Dec 20 10:19:03 CST 2019
mZxid = 0x200000002
mtime = Fri Dec 20 10:19:03 CST 2019
pZxid = 0x200000002
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 11
numChildren = 0
```

> 监听值的变化

在 hadoop03、hadoop04 上，监听 `/laiyy` 节点的变化。

```
[zk: localhost:2181(CONNECTED) 1] get /laiyy watch
test_create
cZxid = 0x200000002
ctime = Fri Dec 20 10:19:03 CST 2019
mZxid = 0x200000002
mtime = Fri Dec 20 10:19:03 CST 2019
pZxid = 0x20000000d
cversion = 7
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 11
numChildren = 5
```


在 hadoop02 上，修改 `/laiyy` 节点的值，查看 hadoop03、hadoop04 上的输出
```
[zk: localhost:2181(CONNECTED) 15] set /laiyy 'emmmm'
cZxid = 0x200000002
ctime = Fri Dec 20 10:19:03 CST 2019
mZxid = 0x200000011
mtime = Fri Dec 20 10:36:33 CST 2019
pZxid = 0x20000000d
cversion = 7
dataVersion = 1
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 5
numChildren = 5
```

```
[zk: localhost:2181(CONNECTED) 1] 
WATCHER::

WatchedEvent state:SyncConnected type:NodeDataChanged path:/laiyy
```


***注意：监听方式只有一次有效，即监听到一次变化后，后续的就不再监听了***

## set 

设置节点的具体值

创建 `/laiyy/name` 节点，赋值为 `laiyy`，使用 set 节点重新赋值

```
[zk: localhost:2181(CONNECTED) 12] get /laiyy/name
laiyy
cZxid = 0x20000000d
ctime = Fri Dec 20 10:31:18 CST 2019
mZxid = 0x20000000d
mtime = Fri Dec 20 10:31:18 CST 2019
pZxid = 0x20000000d
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 5
numChildren = 0
```

```
[zk: localhost:2181(CONNECTED) 13] set /laiyy/name 'hahaha'
cZxid = 0x20000000d
ctime = Fri Dec 20 10:31:18 CST 2019
mZxid = 0x20000000e
mtime = Fri Dec 20 10:32:50 CST 2019
pZxid = 0x20000000d
cversion = 0
dataVersion = 1
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 6
numChildren = 0

[zk: localhost:2181(CONNECTED) 14] get /laiyy/name         
hahaha
cZxid = 0x20000000d
ctime = Fri Dec 20 10:31:18 CST 2019
mZxid = 0x20000000e
mtime = Fri Dec 20 10:32:50 CST 2019
pZxid = 0x20000000d
cversion = 0
dataVersion = 1
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 6
numChildren = 0
```

## stat

查看节点状态

```
[zk: localhost:2181(CONNECTED) 0] stat /laiyy
cZxid = 0x200000002
ctime = Fri Dec 20 10:19:03 CST 2019
mZxid = 0x200000011
mtime = Fri Dec 20 10:36:33 CST 2019
pZxid = 0x200000018
cversion = 9
dataVersion = 1
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 5
numChildren = 7
```

## delete 

删除节点

```
[zk: localhost:2181(CONNECTED) 1] ls /laiyy
[address, gender0000000002, name, gender0000000001, gender0000000004, email, gender0000000003]
[zk: localhost:2181(CONNECTED) 2] delete /laiyy/address
[zk: localhost:2181(CONNECTED) 3] ls /laiyy
[gender0000000002, name, gender0000000001, gender0000000004, email, gender0000000003]
[zk: localhost:2181(CONNECTED) 4] delete /laiyy
Node not empty: /laiyy
```

## rmr

递归删除节点

```
[zk: localhost:2181(CONNECTED) 5] rmr /laiyy
[zk: localhost:2181(CONNECTED) 6] ls /  
[zookeeper]
```