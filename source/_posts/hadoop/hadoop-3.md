---
title: Hadoop（3） <br/> yarn
date: 2019-9-20 17:01:31
updated: 2019-9-20 17:01:31
categories:
    hadoop
tags: [hadoop, linux]
---


在上例中，测试了 HDFS 以及在 HDFS 下执行 MapReduce 示例。本例中，测试启动 YARN，并在 YARN 中执行 MapReduce 程序

<!-- more -->

# 配置 YARN

## 配置 yarn-env.sh

修改 yarn-env.sh 中的 JAVA_HOME

```
vim etc/hadoop/yarn-env.sh
```

将第 23 行的注释去掉，修改 JAVA_HOME 为 `export JAVA_HOME=/opt/module/jdk1.8.0_144`

```
 22 # some Java parameters
 23 # export JAVA_HOME=/home/y/libexec/jdk1.6.0/
 24 if [ "$JAVA_HOME" != "" ]; then
 25   #echo "run java in $JAVA_HOME"
 26   JAVA_HOME=$JAVA_HOME
 27 fi
```

## 配置 yarn-site.xml

```xml
vim etc/hadoop/yarn-site.xml


<configuration>
    <!-- 修改 reduce 获取数据的方式（洗牌） -->
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <!-- 修改 resourcemanager 的 地址  -->
    <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>hadoop01</value>
    </property>
</configuration>
```

## 配置 mapred-env.sh

修改 mapred-env.sh 的 JAVA_HOME：第 16 行去掉注释，修改 JAVA_HOME 为 `export JAVA_HOME=/opt/module/jdk1.8.0_144`

## 创建并修改 mapred-site.xml 文件

拷贝 `etc/hadoop/mapred-site.xml.template` 文件为 `mapred-site.xml` 文件，指定 MR 运行在 YARN 上
```xml
cp mapred-site.xml.template mapred-site.xml
vim mapred-site.xml


<configuration>
    <!-- 指定 MR 运行在 YARN 上  -->
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
</configuration>
```

# 启动 YARN

在保证 NameNode 和 DataNode 启动的情况下，启动 ResourceManager 和 NodeManager。

进入 `/opt/module/hadoop-2.7.2`，利用 `yarn-daemon.sh` 启动。 注意启动顺序：ResourceManager 先启动

```
sbin/yarn-daemon.sh start resourcemanager

sbin/yarn-daemon.sh start nodemanager
```

查看是否启动成功

```
[root@hadoop01 hadoop-2.7.2]# jps
1345 NameNode
1505 DataNode
2068 ResourceManager
2327 NodeManager
```

## 查看 Web UI

访问 hadoop01:50070，hdfs 正常使用，继续访问 hadoop01:8088，查看 MapReduce 程序运行进程

50070：HDFS
8088：MapReduce

![map reduce 进程](/images/hadoop/map-reduce-8088.png)

## 测试 8088 的 MapReduce

删除 hdfs 中的 /user/laiyy/output： ` bin/hdfs dfs -rm -r /user/laiyy/output`

执行 MapReduce word count

```
hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.2.jar wordcount /user/laiyy/input /user/laiyy/output
```

控制台打印日志如下：
```

...

19/09/20 17:35:36 INFO mapreduce.Job: The url to track the job: http://hadoop01:8088/proxy/application_1568971486858_0001/
19/09/20 17:35:36 INFO mapreduce.Job: Running job: job_1568971486858_0001
19/09/20 17:35:46 INFO mapreduce.Job: Job job_1568971486858_0001 running in uber mode : false
19/09/20 17:35:46 INFO mapreduce.Job:  map 0% reduce 0%
19/09/20 17:35:52 INFO mapreduce.Job:  map 100% reduce 0%
19/09/20 17:35:58 INFO mapreduce.Job:  map 100% reduce 100%
19/09/20 17:35:59 INFO mapreduce.Job: Job job_1568971486858_0001 completed successfully
```

> 在什么地方执行的任务：The url to track the job: http://hadoop01:8088/proxy/application_1568971486858_0001/
> map、reduce 执行流程：map x% reduce x%
> 执行结果：Job job_1568971486858_0001 completed successfully

在 8088 上查看执行信息
![yarn map reduce](/images/hadoop/yarn-map-reduce.png)