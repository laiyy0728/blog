---
title: hadoop（14） Map Reduce <BR /> 排序、合并
date: 2019-12-09 16:02:42
updated: 2019-12-09 16:02:42
categories:
    [hadoop, map-reduce, sort]
tags:
    [hadoop, map-reduce, sort]
---

排序，是 MapReduce 中最重要的操作之一。默认的排序方式是 `字典排序`，且实现此排序的方式是 `快速排序`。

MapTask 和 MapReduce 均会对数据按照 `key` 进行排序，该操作属于 Hadoop 的默认操作。任何程序中的数据均会被排序，不论逻辑上是否需要。

<!-- more -->

# 排序概述
> 对于 MapTask

它会将处理的结果暂时放到 `环形缓冲区` 中，当缓冲区的使用率达到一定的阈值之后，再对缓冲区中的数据进行一次快速排序，并将这些有序数据溢写到磁盘上，而当数据处理完毕后，它会对磁盘上的文件进行 `归并排序`。

> 对于 ReduceTask

它从 `每个 MapTask` 上远程拷贝相应的数据文件，如果文件大小超过一定阈值，则溢写到磁盘上，否则存储到内存。
如果磁盘上的文件数据达到一定阈值，则进行一次 `归并排序`，以生成一个更大的文件。
如果内存中文件大小或数目，达到一定阈值，则进行一次 `合并`，将数据溢写到磁盘上。
当所有数据拷贝完成后，ReduceTask 统一对内存和磁盘上的数据进行一次 `归并排序`。

# 排序的分类

> 部分排序

MapReduce 根据输入记录的键，对数据集排序，保证输出的每个文件，内部有序

> 全排序

最终输出结果只有一个文件，且文件内部有序。实现方式是：只设置一个 ReduceTask。但是该犯法在处理大型文件是，效率极低，完全丧失了 MapReduce 所提供的并行架构。

> 辅助排序（GroupingComparator 分组）

在 Reduce 端对 key 进行分组。应用于：在接收的 key 为 bean 对象时，想让一个或几个字段相同（并不是全部字段相同）的 key 进入到同一个 Reduce 方法时，可以使用分组排序

> 二次排序

在自定义排序过程中，如果 compareTo 中的条件为两个，即为二次排序。


## 自定义排序（全排序）

bean 对象作为 key 传输，需要实现 WritableComparable 接口，重写 compareTo 方法。

需求：使用之前 [序列化实例](/hadoop/map-reduce/hadoop-10.html#序列化-Demo) 的结果作为 [输入数据](/file/hadoop/shuffle/sort.txt)，期望输出：按照总流量倒序排序。

### 实现

> Bean

[序列化实例](/hadoop/map-reduce/hadoop-10.html#序列化-Demo) 中的 FlowBean 类，实现 `WritableComparable` 接口，重写 `compareTo` 方法
```java
public int compareTo(FlowBean flowBean) {
    // 比较
    if (sumFlow > flowBean.getSumFlow()){
        return -1;
    } else if (sumFlow < flowBean.getSumFlow()){
        return 1;
    }
    return 0;
}
```

> Mapper

```java
private FlowBean flowBean = new FlowBean();

@Override
protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
    String line = value.toString();

    String[] fields = line.split("\t");

    String phone = fields[0];

    long upFlow = Long.parseLong(fields[1]);
    long downFlow = Long.parseLong(fields[2]);
    flowBean.set(upFlow, downFlow);

    Text text = new Text();
    text.set(phone);
    context.write(flowBean, text);
}
```

> Reducer

```java
@Override
protected void reduce(FlowBean key, Iterable<Text> values, Context context) throws IOException, InterruptedException {

    for (Text value : values) {
        context.write(value, key);
    }

}
```

> Driver 省略，测试结果

![排序结果](/images/hadoop/shuffle/sort.png)


## 区内排序

需求：使用 [输入数据](/file/hadoop/shuffle/sort_1.txt)，期望输出：根据手机号的前三位不同，分成不同的文件，并在每个文件中，按照总流量倒序输出。

在上一例全排序的基础上，增加一个分区操作。

```java
public class FlowBeanPartitioner extends Partitioner<FlowBean, Text> {
    @Override
    public int getPartition(FlowBean flowBean, Text text, int numPartitions) {
        int partitioner = 4;
        String phonePrefix = text.toString().substring(0, 3);
        if ("134".equals(phonePrefix)){
            partitioner = 0;
        } else if ("135".equals(phonePrefix)){
            partitioner = 1;
        } else if ("136".equals(phonePrefix)){
            partitioner = 2;
        }else if ("137".equals(phonePrefix)){
            partitioner = 3;
        }
        return partitioner;
    }
}
```

> 查看结果

![分区排序](/images/hadoop/shuffle/shuffle-sort.png)

---

# 合并

> Combiner 是 MapReduce 程序中 Maper 和 Reducer 之外的一种组件
> Combiner 是父类是 Reducer
> Combiner 与 Reducer 的区别在于运行的位置：Combiner 是在每个 MapTask 所在的节点运行；Reducer 是接收全局所有 Mapper 的输出结果
> Combiner 的意义是对每个 MapTask 的数据进行局部汇总，以减小网络的传输量
> 应用前提：不能影响最终的业务逻辑（Combiner 输出的 KV，应该与 Reducer 的输入 KV 类型对应）

## 自定义 Combiner

需求： 使用 [输入数据](/file/hadoop/combiner/combiner.txt)，进行 `局部汇总`，以减小网络传输量。

期望输出：分隔单词，局部汇总每个单词的数量

***以 WordCount 实例为例***

原始的 WordCount 的控制台输出为：
```
Map-Reduce Framework
    Map input records=8
    Map output records=18
    Map output bytes=168
    Map output materialized bytes=210
    Input split bytes=100
    Combine input records=0
    Combine output records=0
    Reduce input groups=7
    Reduce shuffle bytes=210
    Reduce input records=18
    Reduce output records=7
    Spilled Records=36
    Shuffled Maps =1
    Failed Shuffles=0
    Merged Map outputs=1
    GC time elapsed (ms)=7
    Total committed heap usage (bytes)=514850816
```

> 方案 1

自实现一个 Combiner，并在 Driver 中注册。

```java
public class WordCountCombiner extends Reducer<Text, IntWritable, Text, IntWritable> {

    private IntWritable outValue = new IntWritable();
    
    @Override
    protected void reduce(Text key, Iterable<IntWritable> values, Context context) throws IOException, InterruptedException {
        int sum = 0;
        // 1. 累加求和
        for (IntWritable value : values) {
            sum += value.get();
        }
        outValue.set(sum);
        context.write(key, outValue);
    }
}
```

```java
job.setCombinerClass(WordCountCombiner.class);
```

测试运行：
```
Map-Reduce Framework
    Map input records=8
    Map output records=18
    Map output bytes=168
    Map output materialized bytes=85  //缩小了
    Input split bytes=100
    Combine input records=18    // 相比之前 Combine 有变化
    Combine output records=7
    Reduce input groups=7
    Reduce shuffle bytes=85      // 缩小了
    Reduce input records=7      // 缩小了
    Reduce output records=7
    Spilled Records=14          // 缩小了
    Shuffled Maps =1
    Failed Shuffles=0
    Merged Map outputs=1
    GC time elapsed (ms)=11      // 缩小了
    Total committed heap usage (bytes)=514850816
```

> 方案 2

直接将之前的 Reducer 作为 Combiner 即可。`job.setCombinerClass(WordCountReducer.class);`