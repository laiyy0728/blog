---
title: hadoop（14） Map Reduce <BR /> 排序
date: 2019-12-09 16:02:42
updated: 2019-12-09 16:02:42
categories:
    [hadoop, map-reduce, shuffle]
tags:
    [hadoop, map-reduce, shuffle]
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

