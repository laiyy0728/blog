---
title: Hadoop（8） <br/> DataNode
updated: 2019-12-02 09:56:18
categories:
    [hadoop]
tags: [hadoop]
---

在了解了 NameNode、SecondaryNameNode 的工作机制、FsImage 与 Edits 数据备份、NameNode 安全机制与多目录后，对 NameNode 有了一些基础了解。在此基础之上，接下来了解一下 DataNode 的工作机制。

<!-- more -->

# DataNode 工作机制

DataNode 中存储的每个数据块，以文件形式存储在磁盘上。包括 2 个文件：数据本身、元数据(包括数据块的长度、校验和、时间戳)

1. DataNode 启动后，向 NameNode 注册
2. NameNode 注册成功后，会同步在 NameNode 元数据中
3. DataNode 每个一个`固定的周期`向 NameNode 再注册（默认1小时）
4. DataNode 与 NameNode 以 `3 秒一次` 的心跳机制判断是否断开。心跳返回结果带有 NameNode 给 DataNode 的命令
5. NameNode 超过 `10 分钟`没有收到心跳，则认为该节点不可用

![DataNode 工作机制](/images/hadoop/client/datanode-work.png)

---

# DataNode 数据完整性

1. 当 DataNode 读取 Block 时，它会计算 CheckSum(校验和)
2. 如果计算后的 CheckSum 与 Block 创建时的值不一样，说明 Block 已经损坏
3. Client 读取其他 DataNode 上的 Block
4. DataNode 在其文件创建后，周期性的验证 CheckSum

![DataNode 数据完整性](/images/hadoop/client/check-sum.png)

---

# 掉线时限

DataNode 进程死亡或者网路故障，造成 DataNode 与 NameNode 无法通信，NameNode 不会立即把该节点判定为 `死亡`，要经过一段时间后才会判定为死亡，这段时间称为 `超时时长`。HDFS 默认的超时时长为 10min + 30s

超时时长计算公示： `2 * dfs.namenode.heartbeat.recheck-interval + 10 * dfs.heartbeat.interval`

具体可参考 [官方文档](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/hdfs-default.xml)，默认值为： `dfs.namenode.heartbeat.recheck-interval` 5 分钟(300000)、`dfs.heartbeat.interval` 3秒（3s）

如果需要修改掉线时限，可以修改 `hdfs-site.xml` 文件。

---

# 服役新数据节点

现在已经有 3 台服务器构成了一个 hadoop 集群，如果现在需要在此基础上，再增加一个新的数据节点，就称为 `新数据节点服役`。

> 环境准备

1. 以 hadoop04 为主，克隆一台 hadoop05
2. 修改 hadoop05 的 ip、主机名称
3. 删除原来 HDFS 中的 `data/` 和 `log/`
4. 应用 `/etc/profile` 配置文件

> 服役新节点

1. 启动原始集群 02、03、04 
2. 删除 05 的 `data/` 和 `log/`
3. 启动 05 的 DataNode(`sbin/hadoop-daemon.sh start datanode`)
4. 启动 05 的 NodeManager(`sbin/yarn-daemon.sh start nodemanager`)
5. 在 05 上上传文件进行测试
6. 查看 WebUI 的 datanodes

![datanodes](/images/hadoop/client/datanodes.png)
![新 DataNode 上传文件](/images/hadoop/client/05-upload-file.png)
![新 DataNode 上传文件](/images/hadoop/client/05-file-block.png)

---

# 退役旧数据节点

## 主机白名单

添加到白名单里的主机节点，都允许访问 NameNode，否则不允许访问。

配置步骤：
> 在 NameNode 的 `%HADOOP_HOME%/etc/hadoop` 目录下创建 `dfs.hosts` 文件，添加白名单（此次不添加 05）
```sh
# dfs.hosts
hadoop02
hadoop03
hadoop04
```

> 在 NameNode 的 `hdfs-site.xml` 文件中增加 dfs.hosts 配置

```xml
<!-- 配置白名单 -->
<property>
    <name>dfs.hosts</name>
    <value>/opt/module/hadoop-2.7.2/etc/hadoop/dfs.hosts</value>
</property>
```

> 分发到 03、04，并刷新 NameNode

```
[root@hadoop02 hadoop]# xsync hdfs-site.xml
[root@hadoop02 hadoop]# hdfs dfsadmin -refreshNodes
Refresh nodes successful
```

> 更新 ResourceManager

```
[root@hadoop03 ~]# yarn rmadmin -refreshNodes
19/12/02 14:26:29 INFO client.RMProxy: Connecting to ResourceManager at hadoop03/192.168.233.132:8033
```

> 查看 WebUI

在执行上述操作的时候，02、03、04、05 都未关闭，也就是说，在执行上述操作之前，在 WebUI 的 DataNodes 中是可以看到 02、03、04、05 四台机器的。
在执行完上述操作后，再次查看 WebUI
![白名单](/images/hadoop/client/white-list.png)

> 如果数据不均衡，可以使用命令来实现集群的再平衡

```
[root@hadoop02 hadoop-2.7.2]# sbin/start-balancer.sh 
starting balancer, logging to /opt/module/hadoop-2.7.2/logs/hadoop-root-balancer-hadoop02.out
Time Stamp               Iteration#  Bytes Already Moved  Bytes Left To Move  Bytes Being Moved
```

此时再查看 `README.TXT` 文件的块信息，由 05 变更到 02 上了。

![数据平衡](/images/hadoop/client/data-balance.png)

## 黑名单退役

在黑名单上的主机会被强制退出。

在测试实现这种情况之前，需要将现场恢复，即将 hdfs-site.xml 中添加的白名单配置先注释掉，并刷新 NameNode 和 ResourceManager，启动 05 上的 DataNode

在 NameNode 的 `%HADOOP_HOME%/etc/hadoop` 目录下，创建 `dfs.hosts.exclude` 文件，并添加 05

修改 NameNode 上的 `hdfs-site.xml` 文件，增加 `dfs.hosts.exclude` 配置

```xml
<!-- 黑名单 -->
<property>
    <name>dfs.hosts.exclude</name>
    <value>/opt/module/hadoop-2.7.2/etc/hadoop/dfs.hosts.exclude</value>
</property>
```

刷新 NameNode、ResourceManager
```
[root@hadoop02 hadoop]# hdfs dfsadmin -refreshNodes
Refresh nodes successful

[root@hadoop03 ~]# yarn rmadmin -refreshNodes
19/12/02 14:26:29 INFO client.RMProxy: Connecting to ResourceManager at hadoop03/192.168.233.132:8033
```

查看 WebUI，提示正在退役中(Decommission In Progress)，此时正在退役的节点会将数据块复制到其他节点上，保证数据的完整性。
![退役节点](/images/hadoop/client/decommission.png)

稍等一会后再刷新页面，提示 05 节点已退役完成：Decommissioned。

如果数据不平衡，可以和白名单一样，使用 balance 命令平衡数据。

> 需要注意的点：

1. 等待退役节点状态为 `Decommissioned`，此时所有的块都已经复制完成，停止该节点及节点资源管理器。注意：如果副本数是 3，服役的节点小于等于 3，是不能退役的！需要修改副本数后才能退役！
2. 不允许白名单和黑名单中存在同一个主机


---

# DataNode 多目录 

修改配置与 NameNode 多目录差不多，也是修改 `hdfs-site.xml`，增加如下配置
```xml
<property>
    <name>dfs.datanode.data.dir</name>
    <value>file:///${hadoop.tmp.dir}/dfs/data1,file:///${hadoop.tmp.dir}/dfs/data2</value>
</property>
```

分发到不同主机上后，重启 hadoop 集群即可。

> 注意：

DataNode 与 NameNode 不同，DataNode 每个目录中存放的数据`不一样`。数据不是副本！

