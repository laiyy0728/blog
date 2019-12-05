---
title: hadoop（12） Map Reduce <BR /> MapReduce 框架原理：InputFormat（二）
date: 2019-12-05 10:29:37
updated: 2019-12-05 10:29:37
categories:
    [hadoop, map-reduce, input-format]
tags:
    [hadoop, map-reduce, input-format]
---

此前了解了 InputFormat 运行时，需要参考的 MapTask 并行度决定机制，以及任务提交的流程，那么接下来就需要深入分析 InputFormat 机制。

<!-- more -->

# FileInputFormat 切片机制

## 切片机制

1. 简单的按照文件的内容长度进行切片
2. 切片大小默认为 Block 大小
3. 切片时不考虑数据集整体，而是针对每一个文件单独切片

## 源码中计算切片大小

Math.max(minSize, Math.min(maxSize, blockSize));

mapreduce.input.fileinputformat.split.minsize=1
mapreduce.input.fileinputformat.split.maxsize=Long.MAX_VALUE

基于此，默认情况下，切片大小 等于 blockSize

## 切片大小的设置

maxsize：切片最大值，参数如果调得比 blockSize 小，则会让切片变小，而且就等于配置的这个参数的值。
minsize：切片最小值，如果调的比 blockSize 大，则可以让切片变得比 blockSize 还大

## 获取切片信息的 API

inputSplit.gePath.getName()：获取切片的文件名称
(FileSplit)content.getInputSplit(); 根据文件类型获取切片信息

---

# CombineTextInputFormat 切片机制

Hadoop 默认的 TextInputFormat 切片机制是对任务按文件规划切片，不管文件多小，都会是一个单独的切片，都会交给一个 MapTask，这样如果有大量小文件，就会产生大量的 MapTask，处理效率极其低下。

CombineTextInputFormat 用于小文件过多的场景，它可以将多个小文件从逻辑上规划到一个切片中，这样多个小文件就可以交给一个 MapTask 处理。

## 最大值设置

CombineTextInputFormat.setMaxInputSplitSize(job, 4194304); // 4 M

这个最大值的设置最好按照实际的小文件大小情况来设置。

## 切片机制

生成切片的过程包括：虚拟存储过程、切片过程 两步。

如：虚拟切片最大值为 4 M，现在有 4 个文件，分别为：a.txt(1.7M)、b.txt(5.1M)、c.txt(3.4M)、d.txt(6.8M)。

虚拟存储过程：
> a：1.7 < 4，划分为 1 块（1.7M）
> b：4 < 5.1 < 2*4；则划分为 2 块（2.55M，2.55M）
> c：3.4 < 4：划分为 1块（3.4M）
> d：4 < 6.8 < 2*4：划分为 2块（3.4M，3.4M）

切片过程：

> 如果虚拟存储的文件大小大于设置好的最大值，则单独形成一个切片
> 否则跟下一个虚拟存储文件进行合并，共同形成一个切片

所以最终会形成 3 个切片：(1.7 + 2.55)M、(2.55 + 3.4)M、(3.4 + 3.4)M


## 案例实操

准备 4 个测试文件：[a.txt(1.7M)](/file/hadoop/input-format/a.txt)、[b.txt(5.1M)](/file/hadoop/input-format/b.txt)、[c.txt(3.4M)](/file/hadoop/input-format/c.txt)、[d.txt(6.8M)](/file/hadoop/input-format/d.txt)。

期望：一个切片，处理4个文件。

### 默认处理

利用这个 4 个文件，运行 WordCount 实例，查看切片个数。

![default split](/images/hadoop/map-reduce/default-split.png)

### 设置虚拟存储

> 设置虚拟存储切片最大值为 4M

在任务提交之前，增加配置：
```java
// 设置任务使用虚拟存储切片
job.setInputFormatClass(CombineTextInputFormat.class);

// 设置虚拟存储切片最大值为 4 M
CombineTextInputFormat.setMaxInputSplitSize(job, 4194304);
// 提交 job
boolean succeed = job.waitForCompletion(true);
```

![三片](/images/hadoop/map-reduce/3-split.png)


> 设置虚拟存储切片最大值为 20M

在任务提交之前，增加配置：
```java
// 设置任务使用虚拟存储切片
job.setInputFormatClass(CombineTextInputFormat.class);

// 设置虚拟存储切片最大值为 20 M
CombineTextInputFormat.setMaxInputSplitSize(job, 20971520);
// 提交 job
boolean succeed = job.waitForCompletion(true);
```

![一片](/images/hadoop/map-reduce/1-split.png)


---

# FileInputFormat 实现类

在运行 MapReduce 程序时，输入的文件包括：基于行的日志文件、二进制格式文件、数据库表等 。针对不同的数据类型，MapReduce 如何读取数据？

FileInputFormat 常见实现：TextInputFormat(文本文件)，KeyValueTextInputFormat（基于 KV 的文本文件），NLineInputFormat（按行处理）、CombineTextInputFormat（小文件处理）、自定义 InputFormat。

![实现类](/images/hadoop/map-reduce/first-sub-class.png)

## TextInputFormat

TextInputFormat 是默认的 FileInputFormat 实现类。按行读取每条记录。key 是该行在整个文件中的字节偏移量（LongWritable 类型）；value 是读取到的该行内容（不包括任何终止符，Text 类型）。

## KeyValueTextInputFormat

每一行为一条记录，被分隔符分隔为 key、value。可以通过 `configuration.set(KeyValueLineRecordReader.KEY_VALUE_SEPERATOR, "\t");` 来设定分隔符，默认分隔符是 `\t`。
此时的 key 是每行排在分隔符之前的 Text 序列。

### demo

需求：统计输入文件中，每一行的第一个单词相同的行数。参考文件：[key-value.txt](/file/hadoop/input-format/key-value.txt)

如：输入数据格式为
```
laiyy#dahe.cn like
liyl#dahe.cn like
laiyy#sina.com.cn hate
laiyy#study hadoop
liyl#study hadoop
```

期望输出为：
```
laiyy 3
liyl 2
```

> Mapper

```java
private IntWritable outValue = new IntWritable(1);

@Override
protected void map(Text key, Text value, Context context) throws IOException, InterruptedException {
    System.out.println("当前行的 key ：" + key + " ---> 当前行的 value：" + value);
    context.write(key, outValue);
}
```

> Reducer

```java
private IntWritable outValue = new IntWritable();

@Override
protected void reduce(Text key, Iterable<IntWritable> values, Context context) throws IOException, InterruptedException {
    int sum = 0;
    for (IntWritable value : values) {
        sum += value.get();
    }
    outValue.set(sum);
    context.write(key, outValue);
}
```

> Driver

```java
public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {
    // 获取 job 对象
    Configuration configuration = new Configuration();

    // 设置分隔符
    configuration.set(KeyValueLineRecordReader.KEY_VALUE_SEPERATOR, "#");

    Job job = Job.getInstance(configuration);

    // 省略

    // 设置使用 KeyValue 形式的 InputFormat
    job.setInputFormatClass(KeyValueTextInputFormat.class);

    // 省略
}
```

> 检查输出结果

![kv 输出结果](/images/hadoop/map-reduce/kv-result.png)

## NLineInputFormat

如果使用 NLineInputFormat，代表每个 map 进程处理的 InputSplit 不再按照 Block 去划分，而是按照 NLineInputFormat 指定的行数来划分。
即：`输入文件的总行数/N=切片数`。如果不能整除，切片数为 `商+1`。

如：使用 hadoop 的 [README.txt](/file/hadoop/map-reduce/README.txt) 文件作为输入，如果 N 为 5，则每个输入分片包括 5 行；文件总共 32 行，则应该有 7 个分片。

### Demo

使用 WordCount 实例作为测试代码，对每个单词进行个数统计。根据输入文件的行数来规定输出几个切片。此案例要求每 5 行放入一个切片。

只需要在 WordCount 的 Driver 中，Job 提交之前，加入下列代码即可。Mapper、Reducer 都不需要变动。
```java
// 设置为 NLineInputFormat
job.setInputFormatClass(NLineInputFormat.class);
// 5 行一个切片
NLineInputFormat.setNumLinesPerSplit(job, 5);
```

![NLineInputFormat split](/images/hadoop/map-reduce/nline.png)

---

# 自定义 InputFormat

步骤：
> 自定义一个类，集成 FileInputFormat
> 改写 RecordReader，实现一次读取一个完整文件，封装为 KV
> 在输出的时候，使用 SequenceFileOutputFormat 输出合并文件

无论是 HDFS 还是 MapReduce，在处理小文件时，效率都非常低，但是又难免面临大量小文件的场景。此时，可以使用自定义 InputFormat 实现小文件的合并。

> 需求

将多个小文件，合并为一个 SequenceFile 文件（Hadoop 用来存储二进制形式的 k-v 对的文件格式），SequenceFile 中存储着多着文件，存储形式为 `文件路径 + 名称 为 key，内容为 value`

准备三个小文件：[sf_1.txt](/file/hadoop/input-format/sf_1.txt)，[sf_2.txt](/file/hadoop/input-format/sf_2.txt)，[sf_3.txt](/file/hadoop/input-format/sf_3.txt)

## 自定义 InputFormat

```java
/**
 * Text：key，存储的是文件路径+名称
 * BytesWritable：value，存储的是整个文件的字节流
 */
public class CustomerInputFormat extends FileInputFormat<Text, BytesWritable> {

    @Override
    public RecordReader<Text, BytesWritable> createRecordReader(InputSplit split, TaskAttemptContext context) throws IOException, InterruptedException {

        CustomerRecordReader recordReader = new CustomerRecordReader();
        // 初始化
        recordReader.initialize(split, context);
        return recordReader;
    }
}
```

## 实现 RecordReader

> 自定义实现 RecordReader，实现一次读取一个完整文件，封装为 KV

```java
public class CustomerRecordReader extends RecordReader<Text, BytesWritable> {

    /**
     * 切片
     */
    private FileSplit fileSplit;

    /**
     * 配置信息
     */
    private Configuration configuration;

    /**
     * 输出的 key（路径 + 名称）
     */
    private Text key = new Text();

    /**
     * 输出的 value（整个文件内容）
     */
    private BytesWritable value = new BytesWritable();

    // 标识是否正在读取
    private boolean progressing = true;

    /**
     * 初始化
     * @param split 切片
     * @param context 上下文
     */
    @Override
    public void initialize(InputSplit split, TaskAttemptContext context) throws IOException, InterruptedException {
        this.fileSplit = (FileSplit) split;
        configuration = context.getConfiguration();
    }

    /**
     * 核心业务逻辑
     */
    @Override
    public boolean nextKeyValue() throws IOException, InterruptedException {

        if (progressing) {

            // 获取 fs 对象
            Path path = fileSplit.getPath();
            FileSystem fileSystem = path.getFileSystem(configuration);

            // 获取输入流
            FSDataInputStream inputStream = fileSystem.open(path);

            // 封装 key
            key.set(path.toString());

            // 拷贝，将文件内容拷贝到 buffer 中
            byte[] buffer = new byte[(int) fileSplit.getLength()];
            IOUtils.readFully(inputStream, buffer, 0, buffer.length);

            // 封装 value
            value.set(buffer, 0, buffer.length);

            // 关闭资源
            IOUtils.closeStream(inputStream);

            progressing = false;
            return true;
        }
        return false;
    }

    /**
     * 获取当前的 key
     */
    @Override
    public Text getCurrentKey() throws IOException, InterruptedException {
        return key;
    }

    @Override
    public BytesWritable getCurrentValue() throws IOException, InterruptedException {
        return value;
    }

    /**
     * 获取处理进度
     */
    @Override
    public float getProgress() throws IOException, InterruptedException {
        return 0;
    }

    /**
     * 关闭资源
     */
    @Override
    public void close() throws IOException {

    }
}
```

## MapReduce

> Mapper

```java
public class CustomerMapper extends Mapper<Text, BytesWritable, Text, BytesWritable> {

    @Override
    protected void map(Text key, BytesWritable value, Context context) throws IOException, InterruptedException {
        context.write(key, value);
    }
}
```

> Reducer

```java
public class CustomerReducer extends Reducer<Text, BytesWritable, Text, BytesWritable> {

    @Override
    protected void reduce(Text key, Iterable<BytesWritable> values, Context context) throws IOException, InterruptedException {
        // 循环写出
        for (BytesWritable value : values) {
            context.write(key, value);
        }
    }
}
```

> Driver

```java
public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {
    // 省略
    
    // 设置 InputFormat、OutputFormat
    job.setInputFormatClass(CustomerInputFormat.class);
    job.setOutputFormatClass(SequenceFileOutputFormat.class);

    // 省略
}
```

## 执行结果

![执行结果](/images/hadoop/map-reduce/sf-result.png)