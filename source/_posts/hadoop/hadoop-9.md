---
title: hadoop（9） Map Reduce <BR />  基础概念，WordCount Demo 实现
date: 2019-12-02 15:37:26
updated: 2019-12-02 15:37:26
categories:
    [hadoop, map-reduce]
tags:
    [hadoop, map-reduce]
---

HDFS、MapReduce、Yarn 是 Hadoop 的三大模块，其中，HDFS 负责存储，MapReduce 负责计算，Yarn 负责资源调度

<!-- more -->

# MapReduce

MapReduce 是一个 `分布式运算程序的编程框架`，是用户开发 `基于Hadoop的数据分析应用`的核心框架。

MapReduce 的核心功能，是将 用户编写的业务逻辑代码，和自带的默认组件，整合成一个完整的分布式运算程序，并发的运行在一个 Hadoop 集群上。

## 优缺点

### 优点

> 易于编程

简单地实现一些接口，就可以完成一个分布式程序。这个分布式程序可以分布到大量的廉价 pc 机器上运行。也就是说，写一个分布式程序和写一个串行程序一模一样。 就是因为这个特点，使得MapReduce编程变得非常流行。

> 良好的扩展性

当计算资源不能得到满足时，可以通过简单的扩展机器来扩展计算能力。

> 高容错性

MapReduce 设计初衷，就是使程序能够部署在廉价的 pc 机器上，这就要求它具有很高的容错性。
如：一台机器挂掉了，它可以把上面的计算任务转移到另外一个节点上，不至于这个任务运行失败，而且这个过程不需要人工参与，完全由 Hadoop 内部完成。

> 适合 PB 级以上海量数据离线处理

可以实现上千台服务器集群并发工作，提供数据处理能力。

### 缺点

> 不擅长实时计算

无法像 MySQL 一样，在毫秒级或秒级内返回结果

> 不擅长流式计算

流式计算的输入数据是动态的，而 MapReduce 的输入数据集的静态的，不能动态变化。这是由 MapReduce 自身的设计特点决定的。

> 不擅长 DAG(有向图) 计算

有向图：多个应用程序存在依赖关系，后一个程序的输入是前一个程序的输出。
MapReduce 可以做 DAG 计算，但是不推荐。原因的每个 MapReduce 的输出结果都会写入磁盘，会造成大量的磁盘 IO，导致性能低下。


## 核心思想

MapReduce 运算程序一般分为两个阶段：Map 阶段(分),Reduce 阶段(合)。
Map 阶段的兵法 MapTask，完全并发运行，互不相干。
Reduce 阶段的并发 ReduceTask，完全并发运行，互补相干。但是它们的数据依赖于上一阶段的所有 MapTask 并发实例的输出。
MapReduce 编程模型只能包含一个 Map 阶段和一个 Reduce 阶段，如果业务逻辑复杂，只能多个 MapReduce 程序串行运行。

例如：
![MapReduce思想](/images/hadoop/map-reduce/map-reduce.png)

## MapReduce 进程

一个完整的 MapReduce 程序在分布式运行时有三类实例进程：

> MrAppMaster：负责整个程序的过程调度以及正太协调
> MapTask：负责 Map 阶段的整个数据处理流程
> ReduceTask：负责 Reduce 阶段的整个数据处理流程

## Hadoop 数据序列化类型

| Java 类型 | Hadoop Writable 类型 |
| :-: | :-: | 
| boolean | BooleanWritable |
| byte | ByteWritable |
| int | IntWritable |
| float | FloatWritable |
| long | LongWritable |
| double | DoubleWritable |
| String | Text |
| Map | MapWritable |
| array | ArrayWritable |

## MapReduce 编程规范

MapReduce 编程分为三个部分： Mapper、Reducer、Driver

### Mapper 阶段

1. 用户自定义的 Mapper 要继承自己的父类
2. Mapper 的数据数据是 KV 对形式，KV 类型自定义
3. Mapper 的业务逻辑写在 map() 方法中
4. Mapper 的输出数据是 KV 对形式，KV 类型自定义
5. map() 方法对每个 KV 只调用一次

## Reducer 阶段

1. 用户自定义的 Reducer 要继承自己的父类
2. Reducer 的输入类型对应 Mapper 的输出数据类型，也是 KV
3. Reducer 的业务逻辑写在对应的 reduce() 方法中
4. reduce() 方法对每个 KV 只调用一次

## Driver 阶段

相当于 YARN 集群的客户端，用于提交整个程序到 YARN 集群，提交的是封装了 MapReduce 程序相关运行参数的 Job 对象

---

# WordCount(Demo)

需求：给定一个文本文件 [README.txt](/file/hadoop/map-reduce/README.txt)，统计文本中每个单词出现的总次数

流程分析：

> Mapper 阶段

1. 将 MapTask 传给我们的文本内容转换为 String
2. 根据空格将单词分为单词
3. 将单词输出为 <单词, 1> (如果有相同的单词，输出的也是 <单词, 1>， 因为 map 阶段只做拆分，不做合并)

> Reducer 阶段

1. 汇总各个 key 的个数
2. 输出该 key 的总次数

> Driver

1. 获取配置信息，获取 job 对象实例
2. 指定本程序的 jar 包所在的本地路径
3. 关联 Mapper、Reducer 业务类
4. 指定 Mapper 输出的 KV 类型
5. 指定最终输出的数据 KV 类型
6. 指定 job 的原始文件所在目录
7. 指定 job 的输出结果所在目录
8. 提交 job


## Mapper 实现

```java
/**
 * @author laiyy
 * @date 2019/12/3 16:36
 *
 * LongWritable：输入数据的 key（此处为偏移量） 类型
 * Text：输入数据的 value（每一行数据） 类型
 * Text：输出数据的 key 类型
 * IntWritable：输出数据的 value 类型
 */
public class WordCountMapper extends Mapper<LongWritable, Text, Text, IntWritable> {

    private Text outKey = new Text();
    private IntWritable writable = new IntWritable(1);

    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
        // 1. 将当前读入的行数据转换为 String 类型
        String line = value.toString();
        // 2. 切割单词
        String[] words = line.split(" ");
        // 3. 循环写出
        for (String word : words) {
            outKey.set(word);;
            writable.set(1);
            context.write(outKey, writable);
        }
    }
}
```

## Reducer 实现

```java
/**
 * @author laiyy
 * @date 2019/12/3 16:49
 *
 * Text：Map 阶段输出的 key 类型
 * IntWritable：Map 阶段输出的 value 的类型
 * Text：最终结果的 key 的类型
 * IntWritable：最终结果的 value 类型
 */
public class WordCountReducer extends Reducer<Text, IntWritable, Text, IntWritable> {

    private IntWritable outValue = new IntWritable();
    
    @Override
    protected void reduce(Text key, Iterable<IntWritable> values, Context context) throws IOException, InterruptedException {
        /*
            在 Map 阶段输出的数据格式为： {key1,1}, {key2,1} ,如： {china,1}, {japan,1}, {china, 1}
            在 Reducer 阶段作为输入时， 相同的将合并，即
            key：china，
            values：[1,1]
         */

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

## Driver 实现

```java
public class WordCountDriver {

    public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {
        // 获取 job 对象
        Configuration configuration = new Configuration();
        Job job = Job.getInstance(configuration);

        // 设置 jar 存放路径
        job.setJarByClass(WordCountDriver.class);

        // 关联 Mapper、Reducer 业务类
        job.setMapperClass(WordCountMapper.class);
        job.setReducerClass(WordCountReducer.class);

        // 指定 Mapper 输出的 KV 类型
        job.setMapOutputKeyClass(Text.class);
        job.setMapOutputValueClass(IntWritable.class);

        // 指定最终输出的数据 KV 类型
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(IntWritable.class);

        // 指定 job 的输入文件所在目录
        FileInputFormat.setInputPaths(job, new Path("d:/dev/README.txt"));

        // 指定 job 的输出结果所在目录
        FileOutputFormat.setOutputPath(job, new Path("d:/dev/mr-demo1"));

        // 提交 job，为 true 时打印执行信息
//        job.submit();
        boolean succeed = job.waitForCompletion(true);

        System.exit(succeed ? 0 : 1);
    }

}
```

## 执行测试

运行时可以看到打印信息如下：
![word-count-succeed](/images/hadoop/map-reduce/word-count-demo.png)
![word-count-succeed](/images/hadoop/map-reduce/word-count-demo-succeed.png)

打开执行后的 `part-r-00000` 文件如下：
![word-count-result](/images/hadoop/map-reduce/word-count-result.png)

## 在集群中运行

将 Driver 的输入、输出路径改为参数获取方式：
```java
// 指定 job 的输入文件所在目录
FileInputFormat.setInputPaths(job, new Path(args[0]));

// 指定 job 的输出结果所在目录
FileOutputFormat.setOutputPath(job, new Path(args[1]));
```

在 pom.xml 文件中增加如下配置：
```xml
<build>
    <plugins>
        <plugin>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>2.3.2</version>
            <configuration>
                <source>1.8</source>
                <target>1.8</target>
            </configuration>
        </plugin>

        <plugin>
            <artifactId>maven-assembly-plugin</artifactId>
            <configuration>
                <descriptorRefs>
                    <descriptorRef>jar-with-dependencies</descriptorRef>
                </descriptorRefs>
                <archive>
                    <!-- 打包主类 -->
                    <mainClass>com.laiyy.study.mapreduce.wordcount.WordCountDriver</mainClass>
                </archive>
            </configuration>
            <executions>
                <execution>
                    <id>make-assembly</id>
                    <phase>package</phase>
                    <goals>
                        <goal>single</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

然后使用 `mvn package` 打包后，会生成两个文件：

```
hadoop-1.0-SNAPSHOT.jar
hadoop-1.0-SNAPSHOT-jar-with-dependencies.jar
```

第一个 jar 是没有 hadoop 依赖的，第二个 jar 是有 hadoop 依赖的。

由于在 hadoop 集群上运行，所以可以使用第一个 jar。如果服务器上没有 hadoop 依赖，则使用第二个 jar 即可。

将第一个 jar 上传至 hadoop，由于使用的是 hadoop 集群，所以输入、输出路径均为 HDFS 路径。

运行测试：
```
[root@hadoop02 hadoop-2.7.2]# hadoop jar hadoop-1.0-SNAPSHOT.jar com.laiyy.study.mapreduce.wordcount.WordCountDriver /laiyy /laiyy/output
19/12/04 14:11:07 INFO client.RMProxy: Connecting to ResourceManager at hadoop03/192.168.233.132:8032
19/12/04 14:11:09 WARN mapreduce.JobResourceUploader: Hadoop command-line option parsing not performed. Implement the Tool interface and execute your application with ToolRunner to remedy this.
19/12/04 14:11:10 INFO input.FileInputFormat: Total input paths to process : 3
19/12/04 14:11:10 INFO mapreduce.JobSubmitter: number of splits:3
19/12/04 14:11:10 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1575438331337_0001
19/12/04 14:11:11 INFO impl.YarnClientImpl: Submitted application application_1575438331337_0001
19/12/04 14:11:11 INFO mapreduce.Job: The url to track the job: http://hadoop03:8088/proxy/application_1575438331337_0001/
19/12/04 14:11:11 INFO mapreduce.Job: Running job: job_1575438331337_0001
19/12/04 14:11:24 INFO mapreduce.Job: Job job_1575438331337_0001 running in uber mode : false
19/12/04 14:11:24 INFO mapreduce.Job:  map 0% reduce 0%
...
```

执行结束后，查看 HDFS 中 /laiyy/output 文件夹内容，并下载 `part-r-0000` 文件，查看文件输出.

![WordCount在集群运行结果](/images/hadoop/map-reduce/wordcount-in-cluster.png)
![WordCount在集群运行结果](/images/hadoop/map-reduce/word-count-cluster-result.png)