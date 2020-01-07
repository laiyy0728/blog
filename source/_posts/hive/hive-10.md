---
title: Hive(十) <BR/> 数据查询（2）
date: 2020-01-06 11:28:21
updated: 2020-01-06 11:28:21
categories:
    [hive]
tags:
    [hive]
---

除了基础的查询之外，Hive 还提供了 `分桶及抽样查询`、`CASE WHEN`、`行列互转` 等常用查询方式。

<!-- more -->

# 分桶及抽样查询

分区针对的是数据的存储路径；分桶针对的是数据文件。

分区提供一个隔离数据和优化查询的便利方式。不过，并非所有的数据集都可以形成合理的分区，特别是需要确定合适的划分大小这个疑虑。

分桶是将数据集分解成更容易管理的若干部分的另一个技术。

## 分桶测试

> 创建分桶表

```sql
hive (default)> create table stu_buck(id int, name string)
              > clustered by(id)  -- 根据哪个字段分桶，必须是表中原有字段
              > into 4 buckets  -- 分几个桶，即会将数据分成几个文件
              > row format delimited fields terminated by '\t';
OK
Time taken: 0.194 seconds
```

> 查看表结构

```
hive (default)> desc formatted stu_buck;
OK
col_name	data_type	comment
# col_name            	data_type           	comment             
	 	 
id                  	int                 	                    
name                	string              	                    
	 	 
# 省略

Compressed:         	No                  	 
Num Buckets:        	4                   	 
Bucket Columns:     	[id]                	 
Storage Desc Params:	 	 
	field.delim         	\t                  
	serialization.format	\t                  
Time taken: 0.063 seconds, Fetched: 28 row(s)
```

> 导入 [测试数据](/file/hive/stu_buck.txt)

```
hive (default)> load data local inpath '/opt/module/hive/tmp_data/stu_buck.txt' into table stu_buck;
Loading data to table default.stu_buck
Table default.stu_buck stats: [numFiles=1, totalSize=158]
OK
Time taken: 0.97 seconds
```

> 查看数据是否存在

```
hive (default)> select * from stu_buck;
OK
stu_buck.id	stu_buck.name
1	ss1

...

16	ss16
Time taken: 0.039 seconds, Fetched: 16 row(s)
```

> 查看 WebUI

此时查看 webUi，可以看到，文件只有一个，而不是分桶设置的 4 个文件。原因是 load 命令实际上调用的是 hadoop 的 put 命令，这个命令是不会进行数据分桶的。

![分桶 1](/images/hive/db/buck_1.png)

### 分桶

> 先将 stu_buck 表的数据清空

```
hive (default)> truncate table stu_buck;
OK
Time taken: 0.077 seconds
```

> 创建一个非分桶表

```sql
hive (default)> create table stu(id int, name string)
              > row format delimited fields terminated by '\t';
OK
Time taken: 0.12 seconds
```

> 将数据导入非分桶表

```
hive (default)> load data local inpath '/opt/module/hive/tmp_data/stu_buck.txt' into table stu;
Loading data to table default.stu
Table default.stu stats: [numFiles=1, numRows=0, totalSize=110, rawDataSize=0]
OK
Time taken: 0.212 seconds
```

> 使用 MR 任务，将非分桶表的数据导入分桶表

```
hive (default)> insert into stu_buck
              > select * from stu;
Query ID = root_20200107104239_1fac1d2c-64a1-4bdb-be69-e39bc07b3b27
...

Time taken: 25.352 seconds
```

```
hive (default)> SELECT * FROM stu_buck;
OK
stu_buck.id	stu_buck.name
1	ss1

...

16	ss16
Time taken: 0.061 seconds, Fetched: 16 row(s)
```

![分桶 2](/images/hive/db/buck_2.png)

可以看到，此时还是只有一个分桶。原因是 hive 的分桶设置默认的关闭的。需要修改 hive 的属性。

### 解决方法

> 清空表

```
hive (default)> truncate table stu_buck;
OK
Time taken: 0.062 seconds
```

> 修改属性

```
hive (default)> set hive.enforce.bucketing=true;
hive (default)> set mapreduce.job.reduces=-1;
```

> 重新导入

```
hive (default)> insert into stu_buck 
              > select * from stu;
Query ID = root_20200107104826_96ec5d1b-9587-43d0-b0cf-3b6cfb18ae7e
Total jobs = 1
Launching Job 1 out of 1
Number of reduce tasks determined at compile time: 4

...

Stage-Stage-1: Map: 1  Reduce: 4   Cumulative CPU: 14.83 sec   HDFS Read: 15410 HDFS Write: 398 SUCCESS
Total MapReduce CPU Time Spent: 14 seconds 830 msec
OK
stu.id	stu.name
Time taken: 31.441 seconds
```

![分桶 3](/images/hive/db/buck_3.png)


## 抽样查询

对于非常大的数据，有时用户需要使用的是一个具有代表性的查询结果，而不是全部结果。hive 可以使用对表的抽样查询来满足需求。

语法：`select * from table_name tablesample(bucket x out of y on field_name)`

含义：
> y 必须是 bucket 数的倍数或因子。hive 根据 y 的大小决定抽样比例。如：有 4 个 bucket，当 y 为 2 时，抽取 (4/2=2) 个 bucket 的数据，当 y 为 8 时，抽取 (4/8=1/2) 个 bucket 的数据。
> x 表示从那个 bucket 开始抽取，如果需要取多个分区，以后的分区号为当前分区号加上 y。如：有 4 个bucket，tablesample(bucket 1 out of 2) 表示总共收取 (4/2=2) 个 bucket 的数据，抽取第 1(x) 个和第 3(x+y) 个 bucket 的数据。
> x 必须小于等于 y 的值。

```
hive (default)> select * from stu_buck tablesample(bucket 1 out of 4 on id);
OK
stu_buck.id	stu_buck.name
16	ss16
12	ss12
8	ss8
4	ss4
Time taken: 0.133 seconds, Fetched: 4 row(s)
```

```
hive (default)> select * from stu_buck tablesample(bucket 2 out of 4 on id);
OK
stu_buck.id	stu_buck.name
9	ss9
5	ss5
1	ss1
13	ss13
Time taken: 0.04 seconds, Fetched: 4 row(s)
```

```
hive (default)> select * from stu_buck tablesample(bucket 2 out of 2 on id);
OK
stu_buck.id	stu_buck.name
9	ss9
5	ss5
1	ss1
13	ss13
3	ss3
11	ss11
7	ss7
15	ss15
Time taken: 0.038 seconds, Fetched: 8 row(s)
```

```
hive (default)> select * from stu_buck tablesample(bucket 1 out of 8 on id);
OK
stu_buck.id	stu_buck.name
16	ss16
8	ss8
Time taken: 0.049 seconds, Fetched: 2 row(s)
```

```
hive (default)> select * from stu_buck tablesample(bucket 3 out of 2 on id);
FAILED: SemanticException [Error 10061]: Numerator should not be bigger than denominator in sample clause for table stu_buck
```