---
title: hadoop（15） Map Reduce <BR /> ReduceJoin、MapJoin
date: 2019-12-13 09:43:17
updated: 2019-12-13 09:43:17
categories:
    [hadoop, map-reduce, reduce-join]
tags:
    [hadoop, map-reduce, reduce-join]
---

ReduceJoin 的工作：

Map 端的主要工作：为来自不同表或者文件的 KV 对，打标签以区别不同来源的记录，然后用连接字段作为 key，其余部分和新加的标志位作为 value，最后进行输出。
Reduce 端的主要工作：在 Reduce 端以连接字段作为 key 的分组已经完成，只需要在每个分组中，将那么来源于不同文件的记录分开，最后完成合并即可。

<!-- more -->

# ReduceJoin

## 示例

需求：输入数据为两个表：[订单](/file/hadoop/join/order.txt)、[商品信息](/file/hadoop/join/pd.txt)，将商品信息中的数据，根据商品的 pid，合并到订单数据中。

> TableBean

```java
public class TableBean implements Writable {

    /**
     * 订单 id
     */
    private String id;
    /**
     * 产品 id
     */
    private String pid;
    /**
     * 数量
     */
    private int amount;
    /**
     * 产品名称
     */
    private String pname;
    /**
     * 标记位
     */
    private String flag;

    public void write(DataOutput dataOutput) throws IOException {
        dataOutput.writeUTF(id);
        dataOutput.writeUTF(pid);
        dataOutput.writeInt(amount);
        dataOutput.writeUTF(pname);
        dataOutput.writeUTF(flag);
    }

    public void readFields(DataInput dataInput) throws IOException {
        id = dataInput.readUTF();
        pid = dataInput.readUTF();
        amount = dataInput.readInt();
        pname = dataInput.readUTF();
        flag = dataInput.readUTF();
    }

    @Override
    public String toString() {
        return id + "\t" + pname + "\t" + amount;
    }

    // 省略 get、set
}
```

> Mapper

```java
public class TableMapper extends Mapper<LongWritable, Text, Text, TableBean> {

    private TableBean table = new TableBean();

    private String fileName;

    private Text outKey = new Text();

    /**
     * 在 map 执行前，获取文件名称，来判断当前处理的是哪个文件
     */
    @Override
    protected void setup(Context context) throws IOException, InterruptedException {
        FileSplit inputSplit = ((FileSplit) context.getInputSplit());
        fileName = inputSplit.getPath().getName();
    }

    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
        String line = value.toString();
        String[] fields = line.split("\t");
        if (fileName.contains("order")) {
            // order.txt
            table.setId(fields[0]);
            table.setPid(fields[1]);
            table.setAmount(Integer.parseInt(fields[2]));
            table.setFlag("order");
        } else {
            // pd.txt
            table.setFlag("pd");
            table.setPid(fields[0]);
            table.setPname(fields[1]);
        }
        outKey.set(table.getPid());

        context.write(outKey, table);
    }
}
```

> Reducer

```java
public class TableReducer extends Reducer<Text, TableBean, TableBean, NullWritable> {

    @Override
    protected void reduce(Text key, Iterable<TableBean> values, Context context) throws IOException, InterruptedException {

        List<TableBean> tableBeans = Lists.newArrayList();
        // 注意！！！！！
        // 一定要遍历中重新创建对象，拷贝数据到新创建的对象中，并把新创建的对象放入一个新的 list 中！！
        // 否则 tableBeans 里面的对象都是同一个对象！！！
        for (TableBean value : values) {
            TableBean tableBean = new TableBean();
            try {
                // 示例使用的是 commons 包的对象拷贝，阿里规范禁止使用，如果在 spring 项目中使用 hadoop，尽量使用 spring util 的对象拷贝
                BeanUtils.copyProperties(tableBean, value);
                tableBeans.add(tableBean);
            } catch (IllegalAccessException | InvocationTargetException e) {
                e.printStackTrace();
            }
        }

        // 1. 获取 orders
        List<TableBean> orders = tableBeans.stream().filter(tableBean -> tableBean.getFlag().equals("order"))
                .collect(Collectors.toList());
        // 2. 获取商品，并转换为 map，key 为商品id，value 为商品名称
        Map<String, String> pd = tableBeans.stream().filter(tableBean -> tableBean.getFlag().equals("pd"))
                .collect(Collectors.toMap(TableBean::getPid, TableBean::getPname));


        // 3. 遍历 orders，赋值 pname，并输出

        for (TableBean order : orders) {
            order.setPname(pd.get(order.getPid()));

            context.write(order, NullWritable.get());
        }

    }
}
```

> 省略 Driver，查看运行结果

![reduce join ](/images/hadoop/map-reduce/reduce-join.png)

## 缺点

合并的操作在 Reduce 阶段完成，Reduce 端的处理压力太大，Map 节点的运算负载很低，资源利用率不高，而且在 Reduce 阶段极易产生数据倾斜。 推荐使用 MapJoin

---

# MapJoin

适用场景： 一个张表十分小，一个表十分大

> 优点

在 Map 端缓存多张表，提前处理业务逻辑，增加了 Map 业务，减少 Reduce 数据压力，尽可能减少数据倾斜

> 方法

1、在 Mapper 的 setup 阶段，将文件读取到缓存集合中
2、在驱动函数中加载缓存

## 实现

依然使用上例中的输入数据，输入结果也应该与上例一致。


> Driver

***由于 MapJoin 不需要 Reduce 端，此时可以将 Driver 中的 MapperKeyClass、MapperValueClass、ReduceClass 去掉***

```java
public static void main(String[] args) throws Exception {

    // 获取 job 对象
    Configuration configuration = new Configuration();

    Job job = Job.getInstance(configuration);

    // 设置 jar 存放路径
    job.setJarByClass(MapJoinDriver.class);

    // 关联 Mapper、Reducer 业务类
    job.setMapperClass(MapJoinMapper.class);

    // 指定最终输出的数据 KV 类型
    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(NullWritable.class);

    // 指定 job 的输入文件所在目录
    // 只读入 order
    FileInputFormat.setInputPaths(job, new Path("D:\\dev\\join\\order.txt"));

    // 指定 job 的输出结果所在目录
    FileOutputFormat.setOutputPath(job, new Path("D:\\dev\\join\\output1"));

    // 加载缓存数据
    job.addCacheFile(new URI("file:///d:/dev/join/pd.txt"));
    // MapJoin 的逻辑不需要 Reduce，设置 ReduceTask 为 0
    job.setNumReduceTasks(0);

    // 提交 job
    boolean succeed = job.waitForCompletion(true);

    System.exit(succeed ? 0 : 1);

}
```

> Mapper

```java
public class MapJoinMapper extends Mapper<LongWritable, Text, Text, NullWritable> {

    private Map<String, String> fieldMap = new HashMap<>();

    private Text outKey = new Text();

    @Override
    protected void setup(Context context) throws IOException, InterruptedException {
        String path = context.getCacheFiles()[0].getPath();
        BufferedReader reader = new BufferedReader(new InputStreamReader(new FileInputStream(path), StandardCharsets.UTF_8));
        String line;
        while (StringUtils.isNotBlank(line = reader.readLine())){
            String[] fields = line.split("\t");
            fieldMap.put(fields[0], fields[1]);
        }
        IOUtils.closeStream(reader);
    }

    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
        String line = value.toString();
        String[] fields = line.split("\t");

        String pid = fields[1];

        String pname = fieldMap.get(pid);

        outKey.set(fields[0] + "\t" + pname + "\t" + fields[2]);

        context.write(outKey, NullWritable.get());
    }
}
```

> 查看结果

![MapJoin](/images/hadoop/map-reduce/map-join.png)