---
title: hadoop（18） YARN <BR /> 资源调度器、Hadoop 优化
date: 2019-12-16 15:33:32
updated: 2019-12-16 15:33:32
categories:
    [hadoop, yarn]
tags:
    [hadoop, yarn]
---


Yarn 是一个资源调度平台，负责为运算程序提供服务器运算资源，相当于一个分布式的 `操作系统`，而 MapReduce 等运算程序相当于 `操作系统上的应用程序`

<!-- more -->

# 基础架构

Yarn 由 `ResourceManager`、`NodeManager`、`ApplicationMaster`、`Container` 等组件构成。

> ResourceManager

1. 处理客户端请求
2. 监控 NodeManager
3. 启动或监控 ApplicationMaster
4. 资源的分配和调度

> NodeManager

1. 管理单个节点上的资源
2. 处理来自 ResourceManager 的命令
3. 处理来自 ApplicationMaster 的命令


> ApplicationMaster

1. 负责数据的切分
2. 为应用程序申请资源，并分配给内部的任务
3. 任务的监控、容错

> Container

是 YARN 资源的抽象，封装了某个节点上的多维度资源，如：内存、CPU、磁盘、网络等。


---

# 工作机制

1. MapReduce 程序提交任务到客户端所在的节点
2. 申请一个 Application 到 ResourceManager
3. ResourceManager 返回 `资源提交路径` 和 `application_id`
4. 将任务所需资源提交到 `资源提交路径` 上，包括切片信息(Job.Split)，任务信息（Job.xml）、所需 jar 包
5. 资源提交完成后，申请运行 MrAppMaster
6. ResourceManager 将用户的请求初始化为一个 Task，进入任务队列
7. NodeManager 到 ResourceManager 领取 Task 任务
8. NodeManager 创建 Container 容器，分配 CPU、内存等资源，启动对应的 MrAppMaster
9. NodeManager 下载 job 资源到本地，MrAppMaster 读取切片信息，决定开启多少个 MapTask
10. Container 向 ResourceManager 申请运行 MapTask 容器
11. 其余 NodeManager 重复 7-10 步骤，等待任务领取完成
12. 任务领取完成后，MrAppMaster 统一发送程序运行脚本，启动 MapTask
13. 所有 MapTask 执行完成后，由 MrAppMaster 向 ResourceManager 申请对应切片数量的 ReduceTask，进行 reduce 工作
14. ReduceTask 从 MapTask 中获取相应的数据
15. ReduceTask 执行完成后，MrAppMaster 想 ResourceManager 申请注销自己

![yarn](/images/hadoop/yarn/yarn.png)

---

# 资源调度器

资源调度器对应上图中的 ***FIFO调度队列***。

Hadoop 资源调度器主要有三种：`FIFO(队列)`、`Capacity Scheduler(容量调度器)`、`Fair Scheduler(公平调度器)`， 默认资源调度器为 `Capacity Scheduler`

![默认资源调度器](/images/hadoop/yarn/yarn-scheduler-class.png)

## FIFO

按照到达时间，先到先服务；单项执行

当有新的服务器节点资源时，从队列中获取一个任务，从任务重分配一个 Task 给节点进行服务

![FIFO](/images/hadoop/yarn/fifo.png)

## Capacity Scheduler

Hadoop 默认调度器，按照到达时间，先到先服务；并发执行

> 支持多个队列，每个队列可配置一定的资源量，每个队列采用 FIFO 调度策略
> 为防止同一个用户的作业独占队列中的资源，调度器会对同一个用户提交的作业所占资源量进行限制。
> 如果调度器中有三个队列，可以从三个队列的头部取出三个任务并发执行，相比 FIFO 提高了任务的执行速度。


计算方式：
首先，计算每个队列中正在执行的任务数与其应该分得的资源之间的比值，选择一个最小（最闲）的队列；
其次，按照作业优先级和提交时间顺序，同时考虑用户资源量限制和内存限制对队列内的任务进项排序

## Fair Scheduler

按照缺额排序，缺额越大越优先；并发度最高

> 支持多队列、多用户
> 每个队列中的资源量可以配置
> 同一个队列中的作业公平共享队列的所有资源

分配方式：
假设有三个队列：QA、QB、QC，每个队列中的任务按照优先级分配资源，优先级越高分配的资源越多。但是每个任务都会分配到资源，以确保 `公平`。
在资源有限的情况下，每个任务理想情况下获得的资源与实际获得的资源可能存在一定的差距，这个差距就称为 `缺额`。
通一个队列中，任务的资源缺额越大，越先获得资源优先执行。作业是按照缺额的高低来先后执行的，且多个作业同时运行。

# 任务的推测执行

***作业完成时间取决于最慢任务的完成时间***

一个作业由若干个 Map 任务和 Reduce 任务构成，因硬件老化、软件 bug 等，某些任务可能运行的非常慢（如：99% 的Map 都完成了，少数的 Mpa 进度很慢完不成）

解决方案：

为慢的任务启动一个 `备份任务`，同时运行，谁先运行完就采用谁的结果。

> 执行推测任务的前台条件

1. 每个 Task 只能有一个备份任务
2. 当前 Job 已完成的 Task 必须小于 5%
3. 在 mapred-site.xml 中开启推测执行（默认是打开的）

```xml
<property>
    <name>mapreduce.map.speculative</name>
    <value>true</value>
</property>

<property>
    <name>mapreduce.reduce.speculative</name>
    <value>true</value>
</property>
```

> 不能启动推测任务的情况

1. 任务间存在严重的负载倾斜
2. 特殊任务（如向数据库中写数据）

## 推测方法

假设某一时刻，任务 T 的执行进度为 progress，则可通过一定的算法来推测出该任务最终完成的时刻 `endTime`；另一方面，如果此刻为该任务开启一个备份任务，则可以推断出备份任务可能的完成时刻 `endTime1`。则：

```
runTime = (currentTimestamp - taskStartTime) / progress
推测运行时间 = (当前时刻 - 任务启动时刻) / 任务运行比例

endTime = runTIme + taskStartTime
推测结束时刻 = 运行时间 + 任务启动时刻

entTime1 = currentTimestamp + avgRunTime
备份任务推测完成时刻 = 当前时刻 + 运行完成任务的平均时间
```

> MR 总是选择 entTime - endTime1 差值最大的任务，并为之启动备份任务
> 为防止大量任务同时启动备份任务造成资源浪费，MR 为每个作业设置了同时启动备份任务数目的上限
> 推测执行机制实际上采用了经典的优化方案：`以空间换时间`。同时启动多个相同的任务处理相同数据，并让这些任务竞争，以缩短数据处理时间。显然这种方法占用更多的计算资源。在集群资源紧缺的情况下，应合理使用，争取在多用少量资源的情况下，减少作业的计算时间


---

# Hadoop 优化

## MapReduce 速度慢的原因

MapReduce 效率的瓶颈在于：计算机性能、IO 操作优化

计算机性能包括：
> CPU、内存、磁盘监控、网络

IO 操作优化包括：
> 数据倾斜
> Map 和 Reduce 数设置不合理
> Map 运行时间太长，导致 Reduce 等待太久
> 小文件过多
> 大量不可分块的超大文件
> 溢写次数过多
> 归并次数过多

## 优化方法

可以从六个方面考虑优化：数据输入、Map、Reduce、IO、数据倾斜、参数调优

### 数据输入

> 合并小文件

在执行 MR 任务之前，将小文件进行合并，大量小文件会产生大量的 Map 任务，增加 Map 任务装载数，而任务的装载比较耗时，从而导致 MR 运行慢。

解决办法：采用 CombinerTextInputFormat 作为输入，解决输入端大量小文件的问题。

### Map

> 减小溢写次数

通过调整 ***mapred-site.xml*** 文件中的 `mapreduce.task.io.sort.mb` 和 `mapreduce.map.sort.spill.percent` 参数，增大触发溢写的内存上限，减少溢写次数，从而减小磁盘 IO

> 减少合并次数

通过调整 ***mapred-site.xml*** 文件中的 `mapreduce.task.io.sort.factor` 参数，增大合并的文件数目，减少合并次数，从而缩短 MR 处理时间

> 在 Map 之后，不影响业务逻辑的前台下，先进行 Combine 处理，减少 IO（适用于汇总）


### Reduce

> 合理设置 Map、Reduce 数

影响 Map 个数的是 `切片`，影响 reduce 个数的是 `setNumReduceTasks` 方法。
这两个数值都不能设置太小，也不能太大。
太小会导致 Task 等待，处理时间长；太大会导致 Map、Reduce 任务间竞争资源，造成处理超时等错误

> 设置 Map、Reduce 共存

调整 ***mapred-site.xml*** 文件中的 `mapreduce.job.reduce.slowstart.completedmaps` 参数，使 Map 运行到一定程度后，Reduce 也开始运行，减少 Reduce 等待时间。

> 规避使用 Reduce

由于 Reduce 在用于连接数据集的时候会产生大量的网络消耗，如果不需要使用 Reduce，则可以进行规避，减少大量的 shuffle 时间

> 合理设置 Reduce 的 buffer

默认情况下，数据达到一个阈值的时候，Buffer 中的数据就会写入磁盘，然后 Reduce 会从磁盘中获得所有数据。
Buffer 和 Reduce 是没有直接关联的，中间有多次 `写磁盘 -> 读磁盘` 的过程，可以通过调整参数来规避，使得 Buffer 中的一部分数据可以直接输送到 Reduce，减少 IO 开销。

通过调整 ***mapred-site.xml*** 文件中的 `mapreduce.reduce.input.buffer.percent` 配置，默认为 0.0。 当数值大于 0 时，会保留指定比例的内存读 Buffer 中的数据直接交给 Reduce，这样一来，设置 Buffer 需要内存、读取数据需要内存、Reduce 计算也需要内存，如果调整的不合理可能会撑爆服务器，因此需要根据作业运行情况去进行调整。


### IO

> 数据压缩

安装 Snappy 或 LZO，开启数据压缩

> 使用 SequenceFile 二进制文件

### 数据倾斜

数据倾斜包含：数据频率倾斜（某一区域内的数据量远远大于其他区域）、数据大小倾斜（部分记录的大小远远大于平均值）

***解决方案***

> 抽样和范围分区

通过对原始数据进行抽样得到的结果集来预设分区边界值。

> 自定义分区

使用自定义分区，将某些 key 发送给固定的 Reduce 实例，将剩余 key 发送给剩余的 Reduce 实例

> Combine

使用 Combine 可以大量较小数据倾斜，在可能的情况下，Combine 的目的就是聚合并精简数据

> 采用 MapJoin，避免 ReduceJoin


### 参数设置

> mapred-default.xml

| 参数 | 说明 |
| :-: | :-: |
| mapreduce.map.memory.mb | 一个 MapTask 可以使用的资源上线，默认 1024M。如果 MapTask 实际使用的资源超过该值，将被强制杀死 |
| mapreduce.reduce.memory.mb |  一个 ReduceTask 可以使用的资源上线，默认 1024M。如果 ReduceTask 实际使用的资源超过该值，将被强制杀死 |
| mapreduce.map.cpu.vcores | 一个 MapTask 可使用的最大 CPU 数目，默认 1 |
| mapreduce.reduce.cpu.vcores | 一个 ReduceTask 可使用的最大 CPU 数目，默认 1 | 
| mapreduce.reduce.shuffle.parallelcopies | 每个 Reduce 去 Map 中获取数据的并行数，默认 5 |
| mapreduce.reduce.shuffle.merge.percent | Buffer 中的数据达到多少比例开始写入磁盘，默认 0.66 |
| mapreduce.reduce.shuffle.input.buffer.percent | Buffer 大小占 Reduce 可用内存的比例，默认 0.7 |
| mapreduce.reduce.input.buffer.percent | 指定多少比例的内存用来存放 Buffer 中的数据，默认 0.0 |
| mapreduce.task.io.sort.mb | Shuffle 的环形缓冲区大小，默认 100M |
| mapreduce.map.sort.spill.percent | 环形缓冲区溢写的阈值，默认 0.8 |
| mapreduce.map.maxattempts | 每个 MapTask 最大重试次数，超过该值认为 MapTask 失败，默认 4 |
| mapreduce.reduce.maxattempts | 每个 ReduceTask 最大重试次数，超过该值认为 ReduceTask 失败，默认 4 |
| mapreduce.task.timeout | Task 超时时间，如果 Task 在一定时间内没有读取新数据，也没有输出数据，则认为 Task 处于 Block 状态，为防止因为用户程序永远 Block 不退出，则强制设置一个超时时间，默认为 10 分钟 |

> yarn-default.xml

| 参数 | 说明 |
| :-: | :-: |
| yarn.scheduler.minimum-allocation-mb | 应用程序 Container 分配的最小内存，默认 1024 |
| yarn.scheduler.maximum-allocation-mb | 应用程序给 Container 份分配的最大内存，默认 8192 |
| yarn.scheduler.minimum-allocation-vcores | 每个 Container 申请最小 CPU 核数，默认 1 |
| yarn.scheduler.maximum-allocation-vcores | 每个 Container 申请的最大 CPU 核数，默认 4 |
| yarn.nodemanager.resource.memory-mb | 给 Container 分配的最大物理内存，默认 8192 |

## 小文件优化

> 小文件弊端

HDFS 上每个文件都要在 NameNode 上建立一个索引，索引大小约 150 byte，当小文件较多时会产生很多索引文件，一方面会大量占用 NameNode 的内存空间，另一方面就是索引文件过大，使得索引速度变慢。

> 优化方式

1. 在数据采集时，将小文件或小批数据合并为大文件再上传 HDFS
2. 在业务处理前，在 HDFS 上使用 MapReduce 程序，对小文件进行合并
3. 在 MapReduce 处理时，使用 CombineTextInputFormat 提高效率

### 解决方案

> Hadoop  Archive

归档是一个高效的将小文件放入 HDFS 块中的文件存档工具，能工将多个小文件打包为一个 HAR 文件，减少 NameNode 内存使用

> Sequence File

由一系列二进制 KV 组成，如果 key 为文件名，value 为文件内存，则可以将大批小文件合并成一个大文件

> CombineFileInputFormat

新的 InputFormat，用于将多个文件合并为一个单独的 Split，且它会考虑数据的存储位置

> 开启 JVM 重用

对于大量小文件的任务，可以开启 JVM 重用，会减少大约一半的运行时间。
JVM 重用原理：一个 Map 运行在一个 JVM 中，开启后，该 Map 在 JVM 上运行完毕后，JVM 会继续运行其他的 Map。
可以通过修改 ***mapred-site.xml*** 中的 `mapreduce.job.jvm.numtasks` 参数，默认为 1，即运行一个 Task 就销毁 JVM