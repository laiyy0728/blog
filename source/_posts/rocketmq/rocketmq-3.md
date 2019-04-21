---
title: RocketMQ（3） Rocket 集群
date: 2019-04-21 19:29:16
updated: 2019-04-21 19:29:16
categories:
tags:
---

RocketMQ 集群模式分为四种：`单 master`、`多 master`、`多 master 多 slave 异步复制`、`多 master 多 slave 同步双写`

<!-- more -->

# 四种集群模式

## 单 master

风险较大，一旦 broker 宕机或者重启，将导致整个服务部可用。不建议线上环境使用

## 多 master

一个集群全部都是 master，没有 slave

- 优点
配置简单，单个 master 宕机，或者重启未付，对应用没有影响，在磁盘配置为 RAID10 时，即是机器宕机不可恢复的情况，消息也不会丢失（异步刷盘会丢失少量消息，同步刷盘不会丢失消息），性能最高

- 缺点
单个 broker 宕机期间，这台机器上未被消费的消息在机器恢复之前不可订阅，消息的实时性会受到影响。

## 多 master 多 slave 异步复制

每个 master 配置一个 slave，有多对 master slave，HA 采用的是异步复制方式，主备有短暂的消息延迟（毫秒级），master 收到消息后立即向应用返回成功标志，同时向 slave 写入消息。

- 优点
即是磁盘损坏，消息丢失的非常少，且消息的实时性不会受到影响。因为 master 宕机后，消费者仍然可以从 slave 消费，此过程对应用透明，不需要人工干预，性能同多个 master 模式一样

- 缺点
master 宕机，磁盘损坏下，会丢失少量消息

## 多 master 多 slave 同步双写

每个 master 配置一个 slave，有多对 master slave，HA 采用同步双写模式，主备都成功才会返回成功

- 优点
数据与服务都无单点，master 宕机情况下，消息无延迟，服务可用性与数据可用性最高

- 缺点
性能比异步复制低 10% 左右，发送单个 master 的 RT 会略高，主机宕机后，slave 不能自动切换为主机（后续版本会支持）

---

# 一主一从

## 修改 master 配置

进入 `conf/2m-2s-async`，修改文件：`broker-a-s.properties`:
```
rm -rf broker-a-s.properties 
cp broker-a.properties  broker-a-s.properties
```

然后打开 `broker-a-s.properties`，修改：
```
brokerId=1
brokerRole=SLAVE
```

修改两个配置文件的 nameserver 为两个服务器对应的 nameserver 地址，多个地址用英文分号分割

## 修改 slave 配置

将 master 的 `broker-a.properties`、`broker-a-s.properties` 同步过来，在 master 上执行
```
scp broker-a.properties 192.168.52.201:/usr/local/include/mq/rocketmq/conf/2m-2s-async/
scp broker-a-s.properties 192.168.52.201:/usr/local/include/mq/rocketmq/conf/2m-2s-async/
```

## 启动集群

依次启动 master、slave 的 nameserver
```
nohup ./bin/mqnamesrv &
```

在 master 上使用 `broker-a.properties` 启动 broker
```
nohup sh ./bin/mqbroker -c /usr/local/include/mq/rocketmq/conf/2m-2s-async/broker-a.properties > /dev/null 2>&1 &
```

在 slave 上使用 `broker-a-s.properties` 启动 broker
```
nohup sh ./bin/mqbroker -c /usr/local/include/mq/rocketmq/conf/2m-2s-async/broker-a-s.properties > /dev/null 2>&1 &
```

## 验证集群

在 rocketmq-console 中，修改 nameserver 配置：
```
rocketmq.config.namesrvAddr=192.168.52.200:9876;192.168.52.201:9876
```

启动 console，并查看集群属性
![一主一从](/images/rocketmq/1-master-1-slave.png)


## 缺陷

当主节点挂掉后，消息将无法写入

---

# 双主双从

双主双从，异步刷盘，同步复制（生产环境建议采用此方式）

## 集群搭建
准备4份 RocketMQ 环境，修改配置文件 `conf/2m-2s-sync/broker-a.properties`，将 `brokerRole` 改为：`SYNC_MASTER`,`flushDiskType` 改为 `ASYNC_FLUSH`，nameserver 为四台服务器的 nameserver 地址其他与之前 async 的配置一样

修改 `conf/2m-2s-sync/broker-a-s.0properties` 的 `brokerId` 为大于 0 的值，`brokerRole` 为 `SLAVE`，nameserver 为四台服务器的 nameserver 地址。

修改 `conf/2m-2s-sync/broker-b.0properties`、`conf/2m-2s-sync/broker-b-2.0properties`，与 a 的区别在与 `brokerName` 都为 broker-b

## 启动集群

每台机器都启动 nameserveer
`nohup ./bin/mqnamesrv &`

在第一台机器上启动 broker-a
```
nohup sh ./bin/mqbroker -c /usr/local/include/mq/rocketmq/conf/2m-2s-sync/broker-a.properties > /dev/null 2>&1 &
```

在第二台机器上启动 broker-b
```
nohup sh ./bin/mqbroker -c /usr/local/include/mq/rocketmq/conf/2m-2s-sync/broker-b.properties > /dev/null 2>&1 &
```

在第三台机器上启动 broker-a-s
```
nohup sh ./bin/mqbroker -c /usr/local/include/mq/rocketmq/conf/2m-2s-sync/broker-a-s.properties > /dev/null 2>&1 &
```

在第四台机器上启动 broker-b-s
```
nohup sh ./bin/mqbroker -c /usr/local/include/mq/rocketmq/conf/2m-2s-sync/broker-b-s.properties > /dev/null 2>&1 &
```


## 验证集群

修改 rocket-console 的配置：`rocketmq.config.namesrvAddr=192.168.52.200:9876;192.168.52.201:9876;192.168.52.202:9876;192.168.52.203:9876`，启动 console，打开 `集群选项卡`：
![双主双从-同步双写-异步刷盘](/images/rocketmq/2-master-2-slave-sync.png)