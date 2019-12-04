---
title: hadoop（10） Map Reduce <BR />  序列化
date: 2019-12-04 14:17:08
updated: 2019-12-04 14:17:08
categories:
    [hadoop, map-reduce]
tags:
    [hadoop, map-reduce]
---

在 MapReduce 的 [数据序列化类型](/hadoop/map-reduce/hadoop-9.html#Hadoop-数据序列化类型) 中，介绍了几种常见的 Hadoop 序列化类，实现了一个基础的 `WordCount` Demo，使用到了 Long、String、Integer 对应的序列化类，那么接下来就需要了解一下 Hadoop 具体的怎么序列化的。

<!-- more -->

# Hadoop 序列化

## 序列化概述

> 什么是序列化、反序列化

序列化：就是把内存中的对象，转换为`字节序列` 或其他数据传输协议，以便存储到磁盘或网络传输。
反序列化：就是将`收到的字节序列`或`其他数据传输协议`或`磁盘的持久化数据`，转换成内存中的对象。

> 为什么要序列化？

一般来说，`对象` 只能生存在内存中，断电即消失。而且，`对象` 只能由本地进程使用，不能被发送到网络上的另外一台计算机中。
然而，`序列化` 可以存储 `对象`，且可以将对象 `发送到远程计算机`。

> 为什么不用 Java 自身的序列化？

Java 的序列化是一个重量级的框架(Serializable)，一个对象被序列化后，别额外附带很多信息，如：校验信息、Header、继承体系等，不便于在网络中高效传输。基于此，Hadoop 开发了一套属于自己的序列化机制：Writable。

> Hadoop 序列化的特点

1. 紧凑：高效使用存储空间
2. 快速：读写数据的额外开销小
3. 可扩展：随着通信协议的升级而升级
4. 互操作：支持多语言交互

## 自定义实现序列化

实现步骤：
1. 实现 Writable 接口
2. 反序列化时，需要反射调用空参构造函数
3. 重写序列化方法
4. 重写反序列化 方法
5. 反序列化的顺序和序列化的顺序保持一致
6. 重写 toString
7. 实现 Comparable 接口（MapReduce 的 Shuffle 过程要求对 key 必须能排序；当需要排序的时候才做）

---

# 序列化 Demo

需求：根据 [测试文件](/file/hadoop/map-reduce/phone.txt)，统计每个手机号的`上行流量`、`下行流量`、`总流量`。
文件中，倒数第三列为上行流量，倒数第二列为下行流量，最后一列为网络请求状态码。

## 创建统计流量的 Bean 对象

创建一个统计流量的 Bean 对象，并实现序列化操作

```java
public class FlowBean implements Writable {

    /**
     * 上行流量
     */
    private long upFlow;

    /**
     * 下行流量
     */
    private long downFlow;

    /**
     * 总流量
     */
    private long sumFlow;

    /**
     * 空参构造，反射用
     */
    public FlowBean() {
    }


    public void set(long upFlow, long downFlow){
        this.upFlow = upFlow;
        this.downFlow = downFlow;
        this.sumFlow = upFlow + downFlow;
    }

    @Override
    public String toString() {
        return upFlow + "\t" + downFlow + "\t" + sumFlow;
    }

    // 省略 get、set

    /**
     * 序列化
     * @param dataOutput 输入输出
     * @throws IOException 可能异常
     */
    public void write(DataOutput dataOutput) throws IOException {
        dataOutput.writeLong(upFlow);
        dataOutput.writeLong(downFlow);
        dataOutput.writeLong(sumFlow);
    }

    /**
     * 反序列化
     * @param dataInput 输入数据
     * @throws IOException 可能异常
     */
    public void readFields(DataInput dataInput) throws IOException {
        // 必须和序列化方法顺序一致
        upFlow = dataInput.readLong();
        downFlow = dataInput.readLong();
        sumFlow = dataInput.readLong();
    }
}
```

## MapReduce 程序

### Mapper

```java
public class FlowCountMapper extends Mapper<LongWritable, Text, Text, FlowBean> {

    private FlowBean flowBean = new FlowBean();
    private Text outKey = new Text();

    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
        // 1   17319758889 192.168.100.1   www.baidu.com 2481    24685   200

        // 1. 获取一行
        String line = value.toString();
        // 2. 切割
        String[] fields = line.split("\t");
        // 3. 封装对象
        outKey.set(fields[1]);

        int length = fields.length;
        long upFlow = Long.parseLong(fields[length - 3]);
        long downFlow = Long.parseLong(fields[length - 2]);

        flowBean.setUpFlow(upFlow);
        flowBean.setDownFlow(downFlow);

        // 4. 写出
        context.write(outKey, flowBean);
    }
}
```

### Reducer

```java
public class FlowCountReducer extends Reducer<Text, FlowBean, Text, FlowBean> {

    private FlowBean flowBean = new FlowBean();
    
    @Override
    protected void reduce(Text key, Iterable<FlowBean> values, Context context) throws IOException, InterruptedException {
        // 两个相同的手机号的访问记录
        // 11   17319788888 192.168.100.11   www.java1234.com 231    28   200
        // 12   17319788888 192.168.100.12    211    7852   200

        long sumUpFlow = 0;
        long sumDownFlow = 0;

        for (FlowBean value : values) {
            sumUpFlow += value.getUpFlow();
            sumDownFlow += value.getDownFlow();
        }

        flowBean.set(sumUpFlow, sumDownFlow);
        context.write(key, flowBean);
    }
}
```

## Deiver

```java
public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {
    // 获取 job 对象
    Configuration configuration = new Configuration();
    Job job = Job.getInstance(configuration);

    // 设置 jar 存放路径
    job.setJarByClass(FlowCountDriver.class);

    // 关联 Mapper、Reducer 业务类
    job.setMapperClass(FlowCountMapper.class);
    job.setReducerClass(FlowCountReducer.class);

    // 指定 Mapper 输出的 KV 类型
    job.setMapOutputKeyClass(Text.class);
    job.setMapOutputValueClass(FlowBean.class);

    // 指定最终输出的数据 KV 类型
    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(FlowBean.class);

    // 指定 job 的输入文件所在目录
    FileInputFormat.setInputPaths(job, new Path(args[0]));

    // 指定 job 的输出结果所在目录
    FileOutputFormat.setOutputPath(job, new Path(args[1]));

    // 提交 job
//        job.submit();
    boolean succeed = job.waitForCompletion(true);

    System.exit(succeed ? 0 : 1);
}
```

## 运行测试

设置输入输出路径：
![输入输出路径](/images/hadoop/map-reduce/flow-count-args.png)

查看输出结果：
![flow count result](/images/hadoop/map-reduce/flow-count-result.png)