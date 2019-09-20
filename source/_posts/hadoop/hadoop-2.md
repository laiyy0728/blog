---
title: Hadoop（2） <br/> 伪分布式
date: 2019-09-20 11:43:06
updated: 2019-09-20 11:43:06
categories:
    hadoop
tags: [hadoop, linux]
---


配置伪分布式集群，需要注意修改对应的 hdfs 配置文件、JAVA_HOME、副本备份个数等信息。另外在启动集群之前，需要格式化 NameNode（只有第一次启动需要格式化）

<!-- more -->


# 修改配置文件


## 修改 core-site.xml 文件

修改 core-site.xml 中关于 hdfs、数据存储等配置

```
cd /opt/module/hadoop-2.7.2/etc/hadoop
vim core-site.xml
```

将 `configutation` 标签内容修改为：
```xml
<configuration>
    <!-- 修改 hdfs 地址 -->
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://hadoop01:9000</value>
    </property>
    <!-- 修改 hadoop 运行时的数据存储位置，可以不存在，启动时会自动创建 -->
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/opt/module/hadoop-2.7.2/etc/hadoop/data/tmp</value>
    </property>
</configuration>
```

## 修改 hadoop-env.sh

修改 hadoop 的默认 JDK 路径，如果不修改配置，则可能在分布式集群环境下导致 JAVA_HOME 失效


```
vim hadoop-env.sh
```

将 JAVA_HOME 从原来的 `export JAVA_HOME=${JAVA_HOME}`，修改为 `/opt/module/jdk1.8.0_144/`


## 修改 hdfs-site.xml

指定 hdfs 副本的数量为 1 个，默认为 3 个
```xml
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
</configuration>
```

---

# 启动集群

## 格式化 NameNode

第一次启动集群时，需要格式化 NameNode，后面如果再启动时不需要格式化 NameNode

```
cd /opt/module/hadoop-2.7.2

bin/hdfs namenode -format
```

控制台输出：
```
...

19/09/20 15:23:31 INFO namenode.FSNamesystem: Retry cache on namenode is enabled
19/09/20 15:23:31 INFO namenode.FSNamesystem: Retry cache will use 0.03 of total heap and retry cache entry expiry time is 600000 millis
19/09/20 15:23:31 INFO util.GSet: Computing capacity for map NameNodeRetryCache
19/09/20 15:23:31 INFO util.GSet: VM type       = 64-bit
19/09/20 15:23:31 INFO util.GSet: 0.029999999329447746% max memory 966.7 MB = 297.0 KB
19/09/20 15:23:31 INFO util.GSet: capacity      = 2^15 = 32768 entries
19/09/20 15:23:31 INFO namenode.FSImage: Allocated new BlockPoolId: BP-1194915434-192.168.233.130-1568964211854
19/09/20 15:23:31 INFO common.Storage: Storage directory /opt/module/hadoop-2.7.2/etc/hadoop/data/tmp/dfs/name has been successfully formatted.
19/09/20 15:23:31 INFO namenode.NNStorageRetentionManager: Going to retain 1 images with txid >= 0
19/09/20 15:23:31 INFO util.ExitUtil: Exiting with status 0
19/09/20 15:23:31 INFO namenode.NameNode: SHUTDOWN_MSG: 
/************************************************************
SHUTDOWN_MSG: Shutting down NameNode at hadoop01/192.168.233.130
************************************************************/
```

当看到控制台输出 `SHUTDOWN_MSG: Shutting down NameNode at hadoop01/192.168.233.130` 时，即为格式化完成。


## 启动 NameNode、DataNode

启动 NameNode

```
[root@hadoop01 hadoop-2.7.2]# sbin/hadoop-daemon.sh start namenode
starting namenode, logging to /opt/module/hadoop-2.7.2/logs/hadoop-root-namenode-hadoop01.out
```

启动 DataNode
```
sbin/hadoop-daemon.sh start datanode
```

## 验证是否启动成功

> JPS 验证

```
[root@hadoop01 hadoop-2.7.2]# jps
1345 NameNode
1505 DataNode
1583 Jps
```

> Web UI 验证

浏览器访问 hadoop01 的 ip:50070，查看 WebUI 是否可以访问


集群的基础信息：
![hdfs](/images/hadoop/hdfs-1.png)

集群的详细信息：
![hdfs](/images/hadoop/hdfs-2.png)

DataNode 信息：
![hdfs](/images/hadoop/hdfs-3.png)

HDFS 文件管理系统：
![hdfs](/images/hadoop/hdfs-4.png)

---

# HDFS 管理

## 创建一个文件夹

在 hdfs 根目录下再创建其他文件夹以保存文件

```
bin/hdfs dfs -mkdir -p /user/laiyy/input
```

然后在 WebUI 中查看
![hdfs](/images/hadoop/hdfs-5.png)


## 将 wcinput 文件夹里的东西上传到 hdfs 中

```
cd /opt/module/demo

../hadoop-2.7.2/bin/hdfs dfs -put wcinput/wc.input /user/laiyy/input
```

`-put` :上传文件

整体命令：将 wcinput 下的 wc.input 文件，上传到 hdfs 中的 /user/laiyy/input 下

验证上传结果：
```
[root@hadoop01 demo]# ../hadoop-2.7.2/bin/hdfs dfs -ls -R /user/laiyy/input
-rw-r--r--   1 root supergroup         57 2019-09-20 16:20 /user/laiyy/input/wc.input
```
![hdfs](/images/hadoop/hdfs-6.png)

## 使用 hdfs 的文件路径运行 word count 示例

```
hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.2.jar wordcount /user/laiyy/input /user/laiyy/output
```

此时的 `input`、`output` 的路径都是 hdfs 的路径，不是linux的路径。

查看执行结果
```
[root@hadoop01 hadoop-2.7.2]# bin/hdfs dfs -ls -R /user/laiyy
drwxr-xr-x   - root supergroup          0 2019-09-20 16:20 /user/laiyy/input
-rw-r--r--   1 root supergroup         57 2019-09-20 16:20 /user/laiyy/input/wc.input
drwxr-xr-x   - root supergroup          0 2019-09-20 16:28 /user/laiyy/output
-rw-r--r--   1 root supergroup          0 2019-09-20 16:28 /user/laiyy/output/_SUCCESS
-rw-r--r--   1 root supergroup         55 2019-09-20 16:28 /user/laiyy/output/part-r-00000
```
![hdfs](/images/hadoop/hdfs-7.png)


***直接查看 hdfs 中 output 中的执行结果***：
```
[root@hadoop01 hadoop-2.7.2]# bin/hdfs dfs -cat /user/laiyy/output/part-r-00000
hadoop	3
hdfs	1
laiyy	1
laiyy0728	1
mapreduce	1
yarn	1
```


---

# 关于 NameNode 格式化

一定不能经常格式化 NameNode。当需要格式化 NameNode 时，需要先用 jps 命令，查看一下 NameNode 和 DataNode 是否都已经关闭，如果没有关闭，需要关闭 NameNode 和 DataNode。

在关闭 NameNode 和 DataNode 的情况下，删除 `HADOOP_HOME` 下的 `data` 和 `log` 文件夹，然后执行 NameNode 格式化命令。

其中 `data` 文件夹可能不在 `HADOOP_HOME` 下，此时该文件夹在 `HADOOP_HOME/etc/hadoop` 下。

> 为何在 DataNode 存在时不能格式化 NameNode？

在 `data` 文件夹或 `data/tmp/dfs` 文件夹下，有两个文件夹，分别为 `data`、`name`，分别对应 DataNode 和 NameNode。

在 `data/current/` 和 `name/current/` 下，都有一个 `VERSION` 文件，在 `VERSION` 文件中，可以看到对应的信息：

- NameNode
```
namespaceID=1748201392
clusterID=CID-7c41ca27-aaa9-4b85-9966-36af0e361a58
cTime=0
storageType=NAME_NODE
blockpoolID=BP-1270008624-192.168.233.130-1568966288591
layoutVersion=-63
```

- DataNode
```
storageID=DS-9be19535-bada-4b25-9076-f7260386a990
clusterID=CID-7c41ca27-aaa9-4b85-9966-36af0e361a58
cTime=0
datanodeUuid=a4e2c662-26b0-4f7f-8bd4-f6012b54c3e7
storageType=DATA_NODE
layoutVersion=-56

```

对比两个文件，可以看到 DataNode 和 NameNode 中的 `clusterID` 完全一致。

此时，如果由于 DataNode 没有停止，或 `data` 文件夹没有清空，就格式化 NameNode 的话，会导致 NameNode 的 `clusterID` 重新生成， 此时，NameNode 和 DataNode 的 `clusterID` 不一致，导致崩溃。


> DataNode 和 NameNode 在 clusterID 不一致，导致只能同时只能有一个工作的问题 

![format namenode error](/images/hadoop/format-namenode-error.png)