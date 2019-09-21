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

Hadoop 集群需要准备至少三台 Linux 主机，本例使用 VM 模拟三台 CentOS 7 主机。三台主机环境需要一致，都需要关闭防火墙、设置静态 ip、设置主机名称、JDK、Hadoop 等安装

## 虚拟机准备

以 hadoop01 为例，克隆三台虚拟机 `hadoop02`、`hadoop03`、`hadoop04`

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

## 核心配置文件