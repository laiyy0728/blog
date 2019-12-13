---
title: hadoop（17） Map Reduce <BR /> 计数器、压缩
date: 2019-12-13 14:45:52
updated: 2019-12-13 14:45:52
categories:
    [hadoop, map-reduce]
tags:
    [hadoop, map-reduce]
---

Hadoop 为每个作业维护若干个内置计数器，以描述多项指标。
如：记录已处理的字节数和记录数，使用户可以监控已处理的输入数据量和已产生的输出数据量

<!-- more -->

# 计数器

计数器 API

> 枚举方式计数
> 采用计数器组、计数器名称的方式计数
> 计数器结果在程序运行后，在控制台上查看

## 示例

在运行核心业务 MapReduce 之前，往往要先进行数据清洗，清理掉不符合用户要求的数据。清理过程往往只需要运行 Mapper 程序，不需要运行 Reduce 程序

使用一个项目中的日志信息作为输入数据，去除字段长度小于 11 的日志信息

***数据信息敏感，不公开***

```java
public class LogMapper extends Mapper<LongWritable, Text, Text, NullWritable> {

    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
        String line = value.toString();

        boolean result = parseLog(line, context);
        if (result){
            return;
        }

        context.write(value, NullWritable.get());
    }

    private boolean parseLog(String line, Context context) {
        String[] fields = line.split(" ");
        if (fields.length > 11){
            context.getCounter("map", "true").increment(1);
            return true;
        }
        context.getCounter("map", "false").increment(1);
        return false;
    }
}
```

```java
public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {
        // 获取 job 对象
    Configuration configuration = new Configuration();

    Job job = Job.getInstance(configuration);

    // 设置 jar 存放路径
    job.setJarByClass(LogDriver.class);

    // 关联 Mapper、Reducer 业务类
    job.setMapperClass(LogMapper.class);


    // 指定最终输出的数据 KV 类型
    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(NullWritable.class);

    // 指定 job 的输入文件所在目录
    FileInputFormat.setInputPaths(job, new Path("D:\\dev\\log\\*"));

    // 指定 job 的输出结果所在目录
    FileOutputFormat.setOutputPath(job, new Path("D:\\dev\\log\\output"));

    job.setNumReduceTasks(0);

    // 提交 job
    boolean succeed = job.waitForCompletion(true);

    System.exit(succeed ? 0 : 1);
}
```

> 查看结果

数据清洗前：
![数据清洗前](/images/hadoop/shuffle/log-before.png)


数据清洗后：
![数据清洗后](/images/hadoop/shuffle/log-after.png)

***具体更复杂的数据清洗，根据输入数据的不同进行高度定制化，此例只做***