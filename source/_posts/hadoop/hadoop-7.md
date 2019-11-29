---
title: Hadoop（7） <br/> NameNode 和 SecondaryNameNode
date: 2019-11-28 09:24:43
updated: 2019-11-28 09:24:48
categories:
    [hadoop]
tags: [hadoop]
---

此前通过代码了解了 HDFS API 和 I/O 操作，并了解了 HDFS 读写数据的过程，对 HDFS 整体运行过程有了初步了解。接下来就需要了解一下 NN（NameNode）、2NN（SecondaryNameNode） 的区别

<!-- more -->

# NN 和 2NN 的工作机制

如果 NameNode 中的元数据存储在 NameNode 节点的磁盘中，由于要经常进行随机访问，还要响应客户端请求，效率会很低。因此，元数据必须要放在内存中。但是，如果只存储在内存中，一旦断点、服务重启，元数据就会丢失。因此，基于这种情况，产生了在磁盘中备份元数据的 `FsImage`。

在此基础上，当在内存中更新元数据，如果同时更新 FsImage，会导致效率降低；如果不更新，会产生一致性问题，一旦 NameNode 断电，数据就会丢失。
基于这种情况，Hadoop 引入了 `Edits` 文件(只进行追加操作，效率高)。每当元数据有更新或者添加时，就会修改内存中的元数据并追加到 Edits 中。
这样，一旦 NameNode 断电，可以通过 FsImage 和 Edits 合并成一个元数据。

但是这样还有一个问题没有解决，那就是如果长时间添加数据到 Edits 中，会导致该文件数据过大，效率很低，而且，一旦断电，恢复元数据需要的时间很长。
因此，需要定时进行 FsImage 和 Edits 的合并。 如果这个操作由 NameNode 完成，效率又会降低。因此引入了一个新的节点 `SecondaryNameNode`，专门用于 FsImage 和 Edits 的合并。

流程：
1. NameNode 启动时，加载 Edits 和 FsImage 到内存（每个 block 占元数据 150byte）
2. 客户端进行增删改操作时，NameNode 要先记录操作日志，更新Edits，再去进行其他后续请求
3. SecondaryNameNode 请求 NameNode，检查是否触发检查点（触发条件：检查时间到，或 Edits 中的数据满：达到 100W 条）
4. 2NN 请求执行检查点（CheckPoint）
5. NameNode 滚动正在写的 Edits（即从 edits_001 文件，滚动到 edits_002 文件，后续的操作日志将写入 edit_002 中）
6. 将 edits_001 和 FsImage 拷贝到 2NN
7. 2NN 将 FsImage 和 edits_001 加载到内存并合并
8. 2NN 生成新的 FsImage（如：fsimage.checkpoint）
9. 将新生成的 FsImage 拷贝到 NameNonde，并重命名为 fsimage，加载到内存

![NameNode 工作机制](/images/hadoop/client/nn-2nn.png)

----

# 镜像文件和编辑日志 (FsImage、Edits)

在 NameNode 所在的服务器中，查看 fsimage 和 edits 文件(`/opt/module/hadoop-2.7.2/data/tmp/dfs/name/current`)
```
-rw-r--r--. 1 root root    1383 11月 26 11:13 edits_0000000000000000001-0000000000000000019
-rw-r--r--. 1 root root    1816 11月 26 12:13 edits_0000000000000000020-0000000000000000043
-rw-r--r--. 1 root root      42 11月 26 13:13 edits_0000000000000000044-0000000000000000045
-rw-r--r--. 1 root root 1048576 11月 26 13:13 edits_0000000000000000046-0000000000000000046
-rw-r--r--. 1 root root      42 11月 26 14:53 edits_0000000000000000047-0000000000000000048
-rw-r--r--. 1 root root 1048576 11月 26 14:53 edits_0000000000000000049-0000000000000000049
-rw-r--r--. 1 root root     260 11月 26 17:39 edits_0000000000000000050-0000000000000000054
-rw-r--r--. 1 root root 1048576 11月 26 17:39 edits_0000000000000000055-0000000000000000055
-rw-r--r--. 1 root root    1081 11月 27 10:11 edits_0000000000000000056-0000000000000000072
-rw-r--r--. 1 root root      42 11月 27 11:11 edits_0000000000000000073-0000000000000000074
-rw-r--r--. 1 root root 1048576 11月 27 11:48 edits_0000000000000000075-0000000000000000080
-rw-r--r--. 1 root root    1339 11月 28 15:32 edits_0000000000000000081-0000000000000000098
-rw-r--r--. 1 root root      42 11月 28 16:32 edits_0000000000000000099-0000000000000000100
-rw-r--r--. 1 root root 1048576 11月 28 16:32 edits_0000000000000000101-0000000000000000101
-rw-r--r--. 1 root root      42 11月 29 11:21 edits_0000000000000000102-0000000000000000103
-rw-r--r--. 1 root root 1048576 11月 29 11:21 edits_0000000000000000104-0000000000000000104
-rw-r--r--. 1 root root 1048576 11月 29 14:19 edits_inprogress_0000000000000000105
-rw-r--r--. 1 root root     765 11月 29 11:21 fsimage_0000000000000000103
-rw-r--r--. 1 root root      62 11月 29 11:21 fsimage_0000000000000000103.md5
-rw-r--r--. 1 root root     765 11月 29 14:19 fsimage_0000000000000000104
-rw-r--r--. 1 root root      62 11月 29 14:19 fsimage_0000000000000000104.md5
-rw-r--r--. 1 root root       4 11月 29 14:19 seen_txid
-rw-r--r--. 1 root root     207 11月 29 14:19 VERSION
```

## 查看 FsImage 文件

此时，查看 `fsimage_0000000000000000103` 文件，可以看到一些简略信息，详细信息都被二进制编码了。

![cat fsimage](/images/hadoop/client/cat-fsimage.png)

我们可以通过 `hdfs` 的命令，将文件转储为可以看懂的 XML 文件：
```sh
# 参数解释：
# oiv：转储 fsimage 文件
# -p：以何种格式转储
# -i：转储哪个文件
# -o：转储到什么位置
hdfs oiv -p XML -i fsimage_0000000000000000103 -o ~/fsimage.xml
```

执行命令，查看对应文件夹下的 `fsimage.xml` 文件(`cat ~/fsimage.xml`)
```xml
<?xml version="1.0"?>
<fsimage>
    <NameSection>
        <genstampV1>1000</genstampV1>
        <genstampV2>1011</genstampV2>
        <genstampV1Limit>0</genstampV1Limit>
        <lastAllocatedBlockId>1073741834</lastAllocatedBlockId>
        <txid>103</txid>
    </NameSection>
    <INodeSection>
        <lastInodeId>16402</lastInodeId>
        <inode>
            <id>16385</id>
            <type>DIRECTORY</type>
            <name></name>
            <mtime>1574923975056</mtime>
            <permission>root:supergroup:rwxr-xr-x</permission>
            <nsquota>9223372036854775807</nsquota>
            <dsquota>-1</dsquota>
        </inode>
        <inode>
            <id>16398</id>
            <type>FILE</type>
            <name>log.out</name>
            <replication>2</replication>
            <mtime>1574818660432</mtime>
            <atime>1574923642491</atime>
            <perferredBlockSize>134217728</perferredBlockSize>
            <permission>root:supergroup:rw-r--r--</permission>
            <blocks>
                <block>
                    <id>1073741829</id>
                    <genstamp>1006</genstamp>
                    <numBytes>1349614</numBytes>
                </block>
            </blocks>
        </inode>
    </INodeSection>
    <INodeReferenceSection></INodeReferenceSection>
    <SnapshotSection>
        <snapshotCounter>0</snapshotCounter>
    </SnapshotSection>
    <INodeDirectorySection>
        <directory>
            <parent>16385</parent>
        </directory>
    </INodeDirectorySection>
    <FileUnderConstructionSection></FileUnderConstructionSection>
    <SnapshotDiffSection>
        <diff>
            <inodeid>16385</inodeid>
        </diff>
    </SnapshotDiffSection>
    <SecretManagerSection>
        <currentId>0</currentId>
        <tokenSequenceNumber>0</tokenSequenceNumber>
    </SecretManagerSection>
    <CacheManagerSection>
        <nextDirectiveId>1</nextDirectiveId>
    </CacheManagerSection>
</fsimage>
```

## 查看 Edits 文件

使用命令将 edits 文件转储为 xml：`hdfs oev -p XML -i edits_0000000000000000102-0000000000000000103 -o ~/edits.xml`
***oev：转储 edits 文件***

```xml
<?xml version="1.0" encoding="UTF-8"?>
<EDITS>
  <EDITS_VERSION>-63</EDITS_VERSION>
  <RECORD>
    <OPCODE>OP_START_LOG_SEGMENT</OPCODE>
    <DATA>
      <TXID>102</TXID>
    </DATA>
  </RECORD>
  <RECORD>
    <OPCODE>OP_END_LOG_SEGMENT</OPCODE>
    <DATA>
      <TXID>103</TXID>
    </DATA>
  </RECORD>
</EDITS>
```

## 注意

NameNode 通过 `%HADOOP_HOME%/data/tmp/name/current/seen_txid` 文件，来确定下次开机启动的时候，合并哪些 Edits。

---

# CheckPoint 设置

2NN CheckPoint 检查点时间设置，通常情况下，每隔一小时执行一次。

此配置可以参考 [Hadoop 官方文档](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/hdfs-default.xml) 中的 `dfs.namenode.checkpoint.period` 设置项，单位为 秒。

```xml
<property>
    <name>dfs.namenode.checkpoint.period</name>
    <value>3600</value>
</property>
```

CheckPoint 每分钟检查一次操作次数，当操作次数达到 `100万` 时，SecondaryNameNode 执行一次。
```xml
<!-- 设置 CheckPoint 操作此时达到多少时执行 2NN -->
<property>
    <name>dfs.namenode.checkpoint.txns</name>
    <value>1000000</value>
</property>
<!-- 设置多长时间检查一次操作数 -->
<property>
    <name>dfs.namenode.checkpoint.check.period</name>
    <value>60</value>
</property>
```

---

# NameNode 故障处理

NameNode 故障后，可以采用如下两种方式进行数据恢复。

## 将 2NN 中的数据拷贝到 NN 存储数据的目录

***注意：此操作依赖于 2NN 的数据完整性***

步骤：
> kill -9 NameNode
> 删除 NameNode 存储的数据(`%HADOOP_HOME%/data/tmp/name/*`)
> 拷贝 2NN 中的数据到 NN 存储数据目录( 2NN 下的 `%HADOOP_HOME%/data/tmp/namesecondary/*`)
> 重启 NameNode

拷贝方法：
```
scp -r root@hadoop04:/opt/module/hadoop-2.7.2/data/tmp/dfs/namesecondary/* /opt/module/hadoop-2.7.2/data/tmp/dfs/name/
```

单独其中 NameNode： `sbin/hadoop-daemon.sh start namenode`


## 导入检查点

使用 -importCheckpoint 选项，启动 NameNode 守护进程，进而将 2NN 中的数据拷贝到 NameNode 中。

步骤：

1. 修改 NameNode 的 hdfs-site.xml 文件：
```xml
<property>
    <name>dfs.namenode.checkpoint.period</name>
    <value>120</value>
</property>
<property>
    <name>dfs.namenode.name.dir</name>
    <value>your hadoop namenode dir</value>
</property>
```

2. kill -9 NameNode
3. 删除 NameNode 中存储的数据(`%HADOOP_HOME%/data/tmp/name/*`)
4. 如果 2NN 和 NameNode 不在同一个主机节点上，需要将 2NN 存储数据的目录，拷贝到 NN 存储数据的平级目录，并删除 `in_use.lock` 文件。

```
scp -r root@hadoop04:/opt/module/hadoop-2.7.2/data/tmp/dfs/namesecondary/* /opt/module/hadoop-2.7.2/data/tmp/dfs/
cd /opt/module/hadoop-2.7.2/data/tmp/dfs/namesecondary
rm -rf in_use.lock
```

5. 导入检查点数据
```
bin/hdfs namenode -importCheckpoint
```

6. 启动 NameNode
`sbin/hadoop-daemon.sh start namenode`

---

# 集群安全模式

## 安全模式的条件

> NameNode 启动

NameNode 启动时，首先将镜像文件（FsImage）加载到内存中，并执行编辑日志（Edits）中的各项操作。
一旦在内存中成功简历文件系统元数据的映像，则创建一个新的 FsImage 文件和一个空白的编辑日志，此时，NameNode 开始监听 DataNode 的请求。
这个过程中，NameNode 一直处于安全模式，即：NameNode 的文件系统对客户端来说是`只读`的。


> DataNode 启动

系统中的数据块的位置并不是由 NameNode 维护的，而是以`块列表`的形式存储在 DataNode 中 。
在系统的`正常操作期间`，NameNode 会在内存中保留所有块位置的映射信息。
在`安全模式`下，DataNode 会向 NameNode 发送最新的块列表信息，NameNode 了解到足够多的`块位置`信息后，即可搞笑运行文件系统。

> 安全模式退出判断

如果满足 `最小副本条件`，NameNode 将会在 `30s` 后退出安全模式。
所谓 `最小副本条件`：是指在整个文件系统中 99.9% 的块满足最小副本级别（默认值：dfs.replication.min=1）。
在启动一个刚刚格式化的 HDFS 集群时，因为系统中还没有任何块，所以 NameNode 不会进入安全模式。

## 安全模式的命令、语法等

当集群处于`安全模式`，不能执行任何写操作。集群启动完成后，自动退出安全模式。

### 常用命令：

> bin/hdfs dfsadmin -safemode get：查看安全模式状态

```
[root@hadoop02 hadoop-2.7.2]# bin/hdfs dfsadmin -safemode get
Safe mode is OFF
```

> bin/hdfs dfsadmin -safemode enter：进入安全模式

```
[root@hadoop02 hadoop-2.7.2]# bin/hdfs dfsadmin -safemode enter
Safe mode is ON
```
![SafeMode Enter](/images/hadoop/client/safe-mode-enter.png)

此时的 HDFS 文件系统状态为：
![SafeMode FileSystem](/images/hadoop/client/safe-mode-file-system.png)

上传一个文件测试一下：
```
[root@hadoop02 hadoop-2.7.2]# bin/hdfs dfs -put README.txt /
put: Cannot create file/README.txt._COPYING_. Name node is in safe mode.
```

可以发现，上传时已经报错：不能上传文件，因为 NameNode 处于 SafeMode。所以在 HDFS 中没有新上传的文件。

> bin/hdfs dfsadmin -safemode leave：离开安全模式

```
[root@hadoop02 hadoop-2.7.2]# bin/hdfs dfsadmin -safemode leave
Safe mode is OFF
```

此时再次执行上传文件，查看文件系统状态
![SafeMode FileSystem](/images/hadoop/client/safe-mode-leave-file-system.png)


> bin/hdfs dfsadmin -safemode wait：等待安全模式

### 等待安全模式

测试等待安全模式的步骤：

> 先使集群进入到安全模式

```
[root@hadoop02 hadoop-2.7.2]# bin/hdfs dfsadmin -safemode enter
Safe
```

> 在 `%HADOOP_HOME%` 路径下创建一个脚本 `safemode.sh`

```sh
#!/bin/bash

hdfs dfsadmin -safemode wait

hdfs dfs -put /opt/module/hadoop-2.7.2/NOTICE.txt /
```

修改权限：
```
chmod 777 safemode.sh
```

执行脚本，可以看到当前进程阻塞住了，并没有上传 NOTICE.txt 文件。
```
 ./safemode.sh
```


在 xshell 中再打开一个窗口，执行命令：
```
[root@hadoop02 hadoop-2.7.2]# bin/hdfs dfsadmin -safemode leave
Safe mode is OFF
```

再查看之前阻塞的进程，发现已经正常通行，且 NOTICE.txt 文件上传到了 HDFS 中
![SafeMode FileSystem](/images/hadoop/client/safe-mode-wait.png)

---

# NameNode 多目录配置

**NameNode 的本地目录可以配置为多个，且每个目录存在的内容相同，增加了可靠性**
**注意：此方法只是保证了数据的可靠性，并不是保证 NameNode 可靠性，它们对应的依然是一个 NameNode 实例**

> 第 0 步：关闭集群

> 第一步：修改 ddfs-site.xml 文件，增加如下内容，并分发到三台机器上

```xml
<property>
    <name>dfs.namenode.name.dir</name>
    <value>file:///${hadoop.tmp.dir}/dfs/name1,file:///${hadoop.tmp.dir}/dfs/name2</value>
</property>
```

```
 xsync etc/hadoop/hdfs-site.xml
```

> 第二步：删除 data 和 logs 中的所有数据

```
[root@hadoop02 hadoop-2.7.2]# rm -rf data/ logs/
[root@hadoop03 hadoop-2.7.2]# rm -rf data/ logs/
[root@hadoop04 hadoop-2.7.2]# rm -rf data/ logs/
```

> 格式化集群并启动

```
[root@hadoop02 hadoop-2.7.2]# hdfs namenode -format
[root@hadoop03 hadoop-2.7.2]# hdfs namenode -format
[root@hadoop04 hadoop-2.7.2]# hdfs namenode -format
```

查看 `%HADOOP_HOME%/data/dfs` 文件夹：
```
[root@hadoop04 ~]# ll /opt/module/hadoop-2.7.2/data/tmp/dfs/
总用量 0
drwxr-xr-x. 3 root root 21 11月 29 16:52 name1
drwxr-xr-x. 3 root root 21 11月 29 16:52 name2
```

可见两个文件夹都创建成功了。

启动集群：
```
[root@hadoop02 hadoop-2.7.2]# sbin/start-dfs.sh
[root@hadoop03 hadoop-2.7.2]# sbin/start-yarn.sh
```

> 上传文件并测试，可以看到在 name1、name2 中都有对应的文件，且信息完全相同

```
[root@hadoop02 hadoop-2.7.2]# hadoop fs -put NOTICE.txt /
```

![namenode more dir](/images/hadoop/client/more-dir.png)