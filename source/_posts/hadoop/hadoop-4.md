---
title: Hadoop（4） <br/> 完全分布式
date: 2019-9-21 17:01:31
updated: 2019-9-21 17:01:31
categories:
    hadoop
tags: [hadoop, linux]
---


在之前的例子中，完成了一个单机版的 Hadoop + HDFS + YARN 的搭建过程，并成功执行测试成功。
本例中将尝试搭建一个完全分布式的 Hadoop 集群，并验证测试。

<!-- more -->


# Hadoop 集群准备

Hadoop 集群需要准备至少三台 Linux 主机（`hadoop02`、`hadoop03`、`hadoop04`），本例使用 VM 模拟三台 CentOS 7 主机。三台主机环境需要一致，都需要关闭防火墙、设置静态 ip、设置主机名称、JDK、Hadoop 等安装

## 虚拟机准备

以 hadoop01 为源，克隆三台虚拟机 `hadoop02`、`hadoop03`、`hadoop04`

## 编写一个集群分发脚本

需要 linux 中事先安装有 `rsync` 脚本。如果执行命令 `rsync` 提示没有此命令，在 CentOS 下，执行 `yum install -y rsync` 安装即可。


需求：循环复制文件到所有节点的相同目录下

在 hadoop01 的 root 用户家目录中，创建一个 bin 目录用于存放脚本，并创建 `xsync` 脚本文件

```sh
#!/bin/bash

# 获取输入参数的个数，如果没有参数，直接退出
params=$#
if((params==0)); then
echo 'please input param'
exit;
fi

# 获取文件名称
param_1=$1
file_name=`basename $param_1`
echo 'file_name is $file_name'

# 获取上级目录的绝对路径
dir=`cd -P $(dirname $param_1); pwd`
echo 'dir is $dir'

# 获取当前用户名称
user=`whoami`

# 循环 hadoop02-04，同步文件
for((host=2;host<=4;host++)); do
    echo 'rsync file to hadoop0$host'
    rsync -rvl $dir/$file_name $user@hadoop0$host:$dir
done

echo 'rsync file done'
```

需要注意：
1. 执行当前脚本的 user 必须有权限操作当前主机待分发的文件或文件夹；也必须在对应的需要分发的主机上拥有此用户，且有操作对应文件夹的权限。
2. 待分发的文件、文件夹必须使用绝对路径，以保证同步到对应主机的位置也是正确的


> 测试分发脚本

执行 `xsync /root/bin`，在 hadoop02-04 上查看 /root 目录下是否同步了 xsync 脚本

![xsync](/images/hadoop/full-dis/xsync.png)


***注意：如果 xsync 命令识别不了，可以把 xsync 文件放置在 /usr/local/bin 目录下***



# 集群部署

## 集群部署规划

| | hadoop02 | hadoop03 | hadoop04 |
| :-: | :-: | :-: | :-: |
| HDFS | NamdeNode、DataNode | DataNode | SecondaryNameNode |
| YARN | NodeManager | ResourceManager、NodeManager | NodeManager|

需要保证：
> NameNode 和 SecondaryNameNode 不在一台机器上
> ResourceManager 上没有 NameNode 和 SecondaryNameNode，防止内存消耗过大

## 核心配置文件

> core-site.xml

在 `hadoop02`  上，修改 core-site.xml 文件

```xml
<configuration>
    <!-- 修改 hdfs 地址 -->
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://hadoop02:9000</value>
    </property>
    <!-- 修改 hadoop 运行时的数据存储位置，可以不存在，启动时会自动创建 -->
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/opt/module/hadoop-2.7.2/data/tmp</value>
    </property>
</configuration>
```

> HDFS

修改 hadoop-env.sh 中的 JAVA_HOME `export JAVA_HOME=/opt/module/jdk1.8.0_144`

修改 hdfs-site.xml 文件
```xml
<configuration>
    <!-- 副本数量改为3 -->
    <property>
        <name>dfs.replication</name>
        <value>3</value>
    </property>
    <!-- 指定 hadoop 辅助名称节点主机配置（SecondaryNameNode，2nn） -->
    <property>
        <name>dfs.namenode.secondary.http-address</name>
        <value>hadoop04:50090</value>
    </property>
</configuration>
```

> YARN

修改 yarn-env.sh、mapred-env.sh 中的 JAVA_HOME `export JAVA_HOME=/opt/module/jdk1.8.0_144`

修改 yarn-site.xml

```xml
<configuration>
    <!-- 修改 reduce 获取数据的方式（洗牌） -->
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <!-- 修改 resourcemanager 的 地址  -->
    <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>hadoop03</value>
    </property>
    <!-- 开启日志聚集
    <property>
        <name>yarn.log-aggregation-enable</name>
        <value>true</value>
    </property>
     -->
    <!-- 日志保留时间 7 天，单位：秒 
    <property>
        <name>yarn.log-aggregation.retain-seconds</name>
        <value>604800</value>
    </property>
    -->
</configuration>
```

修改 mapred-site.xml 
```xml
<configuration>
    <!-- 指定 MR 运行在 YARN 上  -->
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>

    <!-- 修改历史服务器地址
    <property>
        <name>mapreduce.jobhistory.address</name>
        <value>hadoop01:10020</value>
    </property>
    -->
    <!-- 修改历史服务器 WebUI 地址
    <property>
        <name>mapreduce.jobhistory.webapp.address</name>
        <value>hadoop01:19888</value>
    </property>
    -->
</configuration>
```

> 将修改后的配置文件同步到 03、04 上

`xsync /opt/module/hadoop-2.7.2/etc/hadoop/` 即可

# 集群启动

## 集群单点启动

> 第一次启动需要格式化 NameNode

`bin/hdfs namenode -format`

> 启动

在 `hadoop02` 上执行 `sbin/hadoop-daemon.sh start namenode` 及 `sbin/hadoop-daemon.sh start datanode`

在 `hadoop03` 上执行 `sbin/hadoop-daemon.sh start datanode`

在 `hadoop04` 上执行 `sbin/hadoop-daemon.sh start datanode`

## 集群群起

### ssh 免密登陆

在 `hadoop02` 上使用 `ssh-keygen` 生成秘钥，将生成后的 `id_rsa.pub` 文件中的内容，拷贝到 `hadoop03`、`hadoop04` 中

在 `hadoop02` 中使用 `ssh-copy-id hadoop02`、`ssh-copy-id hadoop03`、`ssh-copy-id hadoop04` 命令拷贝公钥。

如果不将公钥拷贝到本机，使用 `ssh hadoop02` 登陆本机时也需要输入密码。

此时即完成 hadoop02 对 03、04 的免密登陆，以同样的方式，在 03、04 上实现免密登陆


### 群起集群 HDFS

> 配置 slaves

在 hadoop02 中配置修改 `etc/hadoop/slaves` 文件，注意：文件内容不允许有空格，不允许有空行

```
vim etc/hadoop/slaves 

hadoop02
hadoop03
hadoop04
```

> 分发到 03、04 上： ` xsync etc/hadoop/slaves`

> 停止现有的服务，群起服务

在 hadoop02 中，使用命令 `sbin/start-dfs.sh` 群起服务
```
[root@hadoop02 hadoop-2.7.2]# sbin/start-dfs.sh
Starting namenodes on [hadoop02]
hadoop02: starting namenode, logging to /opt/module/hadoop-2.7.2/logs/hadoop-root-namenode-hadoop02.out
hadoop02: starting datanode, logging to /opt/module/hadoop-2.7.2/logs/hadoop-root-datanode-hadoop02.out
hadoop04: starting datanode, logging to /opt/module/hadoop-2.7.2/logs/hadoop-root-datanode-hadoop04.out
hadoop03: starting datanode, logging to /opt/module/hadoop-2.7.2/logs/hadoop-root-datanode-hadoop03.out
Starting secondary namenodes [hadoop04]
hadoop04: starting secondarynamenode, logging to /opt/module/hadoop-2.7.2/logs/hadoop-root-secondarynamenode-hadoop04.out
```

使用 jps 在三台服务器上查看启动情况

### 群起集群 YARN

***注意：*** `由于在 yarn-site.xml 中存在配置 yarn.resourcemanager.hostname 的值为 hadoop03，所以在群起 yarn 时，必须在 hadoop03 上启动，否则会报错`

使用命令 `sbin/start-yarn.sh` 群起 YARN
```
[root@hadoop03 hadoop-2.7.2]# sbin/start-yarn.sh 
starting yarn daemons
starting resourcemanager, logging to /opt/module/hadoop-2.7.2/logs/yarn-root-resourcemanager-hadoop03.out
hadoop02: starting nodemanager, logging to /opt/module/hadoop-2.7.2/logs/yarn-root-nodemanager-hadoop02.out
hadoop04: starting nodemanager, logging to /opt/module/hadoop-2.7.2/logs/yarn-root-nodemanager-hadoop04.out
hadoop03: starting nodemanager, logging to /opt/module/hadoop-2.7.2/logs/yarn-root-nodemanager-hadoop03.out
```

### 查看启动结果

> hadoop02

```
[root@hadoop02 hadoop-2.7.2]# jps
2353 DataNode
2837 NodeManager
2985 Jps
2221 NameNode
```

> hadoop03

```
[root@hadoop03 hadoop-2.7.2]# jps
2544 ResourceManager
1932 DataNode
2829 NodeManager
2991 Jps
```

> hadoop04

```
[root@hadoop04 hadoop-2.7.2]# jps
1745 DataNode
2329 Jps
1804 SecondaryNameNode
2175 NodeManager
```

---

# 集群时间同步

使用 crontab 定时同步时间

## crontab 定时任务

> 基本语法： `crontab [command]`

| command | 功能 |
| :-: | :-: |
| -e | 编辑 crontab 定时任务 |
| -l | 查询 crontab 任务|
| -r | 删除当前用户所有的 crontab 任务 |

> crontab 语法参数

| | 含义 | 范围 |
| :-: | :-: | :-: |
| 第一个 * | 一小时当中的第几分钟 | 0~59 |
| 第二个 * | 一天当中的第几个小时 | 0~23 |
| 第三个 * | 一个月当中的第几天 | 1~31 |
| 第四个 * | 一年当中的第几个月 | 1~12 |
| 第五个 * | 一周当中的星期几 | 0~7(0 和 7 都代表星期日) |

> crontab 特殊符号

| 特殊符号 | 含义 |
| :-: | :-: |
| * | 代表任何时间 |
| , | 代表不连续的时间。如：“0 8,12,16 * * *”，代表每天 8点0分，12点0分，16点0分 执行 |
| - | 代表连续时间范围。如：“0 5 * * 1-6”，代表 周一到周六的凌晨5点0分 执行 |
| */n | 代表每隔多久执行一次。如：“*10 * * * *”，代表每隔 10 分钟执行一次 |


## 时间同步

### 时间服务器(hadoop02)

> 查看 ntp 是否安装

在 hadoop02 中，执行 `rpm -qa | grep ntp`，如果没有任何输出，则代表 ntp 没有安装。没有安装的情况下，执行 `yum install -y ntp` 进行安装。
```
[root@hadoop02 ~]# rpm -qa | grep ntp
ntpdate-4.2.6p5-29.el7.centos.x86_64
ntp-4.2.6p5-29.el7.centos.x86_64
```

> 修改 ntp 配置文件

修改 `/etc/ntp.conf` 文件

- 授权网段 `192.168.x.0-192.168.x.255` 上的机器都可以从 hadoop02 上查询和同步时间
   
打开第 17 行的注释，修改网段即可。

将 `#restrict 192.168.1.0 mask 255.255.255.0 nomodify notrap` 修改为 `restrict 192.168.233.0 mask 255.255.255.0 nomodify notrap`

- 修稿集群在局域网中不使用其他互联网时间

将 `第21~24行` 注释即可。

- 当该节点(hadoop02) 丢失网络连接，依然可用采用本地时间作为时间服务器为集群中的其他节点提供时间同步

在上一步的位置增加如下配置
```conf
server 127.127.1.0
fudge 127.127.1.0 stratum 10
```

***注意***：<br/>
`127.0.0.1` 是本地地址 <br/>
`127.127.1.0` 是回环地址


> 修改 /etc/sysconfig/ntpd 文件

在文件中增加 `SYNC_HWCLOCK=yes`，代表 保证硬件时间与系统时间一起同步

> 启动/重启ntpd

```
[root@hadoop02 ~]# systemctl status ntpd.service
● ntpd.service - Network Time Service
   Loaded: loaded (/usr/lib/systemd/system/ntpd.service; disabled; vendor preset: disabled)
   Active: inactive (dead)  # 可以看到此时未启动


[root@hadoop02 ~]# systemctl start ntpd.service
[root@hadoop02 ~]# systemctl status ntpd.service
● ntpd.service - Network Time Service
   Loaded: loaded (/usr/lib/systemd/system/ntpd.service; disabled; vendor preset: disabled)
   Active: active (running) since 一 2019-09-23 17:04:23 CST; 3s ago # 可以看到此时是运行状态
  Process: 1437 ExecStart=/usr/sbin/ntpd -u ntp:ntp $OPTIONS (code=exited, status=0/SUCCESS)
 Main PID: 1438 (ntpd)
   CGroup: /system.slice/ntpd.service
           └─1438 /usr/sbin/ntpd -u ntp:ntp -g
...
```

设置开机启动：`systemctl enable ntpd.service`

### 其他机器(03、04)

在 03、04 上，使用 crontab 同步 02 的时间。

先随便修改一个时间 `date -s '2018-11-11 11:11:11'`，然后编写 crontab 脚本，每分钟从 hadoop02 上同步时间。等待一分钟后再次查看时间。


---


# hadoop 源码编译

在 apache 的 hadoop 官方网站上，hadoop 源码是 32 位的，当需要 64 位的 hadoop 时，就需要重新编译源码

需要准备：
> 能联网的 CentOS、hadoop 源码包、jdk 64 位，apache-ant、maven、protobuf 序列化框架

本例使用版本：
> hadoop：1.7.2
> apache-ant：1.9.9
> maven：3.0.5
> protobuf：2.5.0
> jdk：1.8.0_144 64 bit

另外需要安装插件：`yum install -y glibc-headers gcc-c++ make cmake openssl ncurses-devel` 

***注意：所有操作必须在 root 用户下完成***

## 安装 protobuf

解压 protobuf-2.5.0 到 /opt/module/，进入 /opt/module/protobuf-2.5.0/ 文件夹，依次执行下列命令：
```
./configure
make
make check
make install
ldconfig
```

修改环境变量，设置 protobuf 的环境到 PATH 中。
```shell
export LD_LIBRARY_PATH=/opt/module/protobuf-2.5.0
export PATH=$PATH:$LD_LIBRARY_PATH
```

验证 protobuf 安装是否成功 `protoc --version`

## 编译源码

执行 `tar -zxf hadoop-2.7.2-src.tar.gz ` 解压，然后执行 `mvn package -Pdist,native -DskipTests -Dtar`，成功后，编译好的 64 位安装包就在 hadoop-2.7.2-src/hadoop-dist/target 下。编译期间报错的话继续在此执行此命令就行。

