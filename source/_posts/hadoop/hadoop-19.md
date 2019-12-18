---
title: hadoop（19）Map Reduce <BR /> 多 Job 串联、Top N
date: 2019-12-18 09:37:41
updated: 2019-12-18 09:37:41
categories:
    [hadoop, map-reduce]
tags:
    [hadoop, map-reduce]
---

在之前的示例中，都是单个 Job 执行 MapReduce 程序，如何进行多 Job 串联？

<!-- more -->

# 多 Job 串联

在有大量的文本（文档、网页）时，如何建立索引？

## 示例

现有三个文档作为输入数据：[a.txt](/file/hadoop/map-reduce/a.txt)、[b.txt](/file/hadoop/map-reduce/b.txt)、[c.txt](/file/hadoop/map-reduce/c.txt)，期望输出文件中某个字符串在哪个文件中，分别出现几次。

### 第一次 MapReduce

```java
// Mapper
public class FirstMapper extends Mapper<LongWritable, Text, Text, IntWritable> {

    private String fileName;
    private Text outKey = new Text();
    private IntWritable outValue = new IntWritable(1);

    @Override
    protected void setup(Context context) throws IOException, InterruptedException {
        FileSplit fileSplit = (FileSplit) context.getInputSplit();
        fileName = fileSplit.getPath().getName();
    }

    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
        String[] fields = value.toString().split(" ");
        for (String field : fields) {
            String k = field + "--" + fileName;
            outKey.set(k);
            context.write(outKey, outValue);
        }
    }
}

// Reducer
public class FirstReduce extends Reducer<Text, IntWritable, Text, IntWritable> {

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
}

// Driver
public static void main(String[] args) throws InterruptedException, IOException, ClassNotFoundException {
    Configuration configuration = new Configuration();

    Job job = Job.getInstance(configuration);

    job.setJarByClass(FirstDriver.class);

    job.setMapperClass(FirstMapper.class);
    job.setReducerClass(FirstReduce.class);

    job.setMapOutputKeyClass(Text.class);
    job.setMapOutputValueClass(IntWritable.class);

    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(IntWritable.class);

    FileInputFormat.setInputPaths(job, new Path("D:\\dev\\hadoop\\job\\*.txt"));

    FileOutputFormat.setOutputPath(job, new Path("D:\\dev\\hadoop\\job\\output"));

    boolean succeed = job.waitForCompletion(true);

    System.exit(succeed ? 0 : 1);
}
```

> 运行结果：
![第一次运行结果](/images/hadoop/map-reduce/job1.png)

### 第二次 MapReduce

```java
// Mapper
public class SecondMapper extends Mapper<LongWritable, Text, Text, Text> {

    private Text outKey = new Text();
    private Text outValue = new Text();

    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
        String line = value.toString();
        String[] split = line.split("--");
        outKey.set(split[0]);
        outValue.set(split[1]);

        context.write(outKey, outValue);
    }
}

// Reducer
public class SecondReducer extends Reducer<Text, Text, Text, Text> {

    private Text outValue = new Text();

    @Override
    protected void reduce(Text key, Iterable<Text> values, Context context) throws IOException, InterruptedException {
        StringBuilder builder = new StringBuilder();
        for (Text value : values) {
            builder.append(value.toString().replace("\t", "-->")).append("\t");
        }

        outValue.set(builder.toString());

        context.write(key, outValue);
    }
}

// Driver
public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {
    Configuration configuration = new Configuration();

    Job job = Job.getInstance(configuration);

    job.setJarByClass(SecondDriver.class);

    job.setMapperClass(SecondMapper.class);
    job.setReducerClass(SecondReducer.class);

    job.setMapOutputKeyClass(Text.class);
    job.setMapOutputValueClass(Text.class);

    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(Text.class);

    FileInputFormat.setInputPaths(job, new Path("D:\\dev\\hadoop\\job\\output\\part-r-00000"));

    FileOutputFormat.setOutputPath(job, new Path("D:\\dev\\hadoop\\job\\output1"));


    boolean succeed = job.waitForCompletion(true);

    System.exit(succeed ? 0 : 1);
}
```

> 运行结果

![第二次运行结果](/images/hadoop/map-reduce/job2.png)

# TopN

将 [排序 Demo](/map-reduce/sort/hadoop-14.html#实现) 的输出结果进行加工，输出总使用量的前 5 位。

## 示例

FlowBean 保持 [排序 Demo](/map-reduce/sort/hadoop-14.html#实现) 中的实现不变，修改 Mapper、Reducer

> Mapper

```java
public class TopnMapper extends Mapper<LongWritable, Text, FlowBean, Text> {

    /**
     * 定义一个 TreeMap，作为存储数据的容器
     */
    private TreeMap<FlowBean, Text> flowMap = new TreeMap<>();

    private FlowBean flowBean ;

    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
        String line = value.toString();

        String[] fields = line.split("\t");

        String phone = fields[0];

        long upFlow = Long.parseLong(fields[1]);
        long downFlow = Long.parseLong(fields[2]);
        // 在 map 中创建对象，否则将只会在 flowMap 中存入同一个对象
        flowBean = new FlowBean();
        flowBean.set(upFlow, downFlow);

        Text v = new Text();
        v.set(phone);

        flowMap.put(flowBean, v);

        // 限制 TreeMap 数量
        if (flowMap.size() > 5) {
            flowMap.remove(flowMap.lastKey());
        }
    }

    @Override
    protected void cleanup(Context context) throws IOException, InterruptedException {
        // 遍历 map，输出数据
        flowMap.forEach((key, value) -> {
            try {
                context.write(key, value);
            } catch (IOException | InterruptedException e) {
                e.printStackTrace();
            }
        });
    }
}
```

> Reducer

```java
public class TopnReducer extends Reducer<FlowBean, Text, Text, FlowBean> {

    /**
     * 定义一个 TreeMap，作为存储数据的容器
     */
    private TreeMap<FlowBean, Text> flowMap = new TreeMap<>();

    @Override
    protected void reduce(FlowBean key, Iterable<Text> values, Context context) throws IOException, InterruptedException {
        for (Text value : values) {
            FlowBean flowBean = new FlowBean();
            flowBean.set(key.getUpFlow(), key.getDownFlow());

            flowMap.put(flowBean, new Text(value));

            // 如果超过 10 条，去掉流量最小的一条
            if (flowMap.size() > 5) {
                flowMap.remove(flowMap.lastKey());
            }

        }
    }

    @Override
    protected void cleanup(Context context) throws IOException, InterruptedException {
        // 输出数据
        flowMap.forEach((key, value) -> {
            try {
                context.write(value, key);
            } catch (IOException | InterruptedException e) {
                e.printStackTrace();
            }
        });
    }
}
```

> 省略 Driver，查看结果

![top n](/images/hadoop/map-reduce/topn.png)