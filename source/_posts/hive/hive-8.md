---
title: Hive(八) <BR/> DML
date: 2019-12-31 10:53:36
updated: 2019-12-31 10:53:36
categories:
    [hive, dml]
tags:
    [hive, dml]
---

DDL 是数据定义语言，DML 是数据操作语言。

<!-- more -->

# 数据导入

## load 方式

语法： `load data [local] inpath '' [overwrite] into table table_name [partition(partition_name=partition_value)]`

> load data：表示加载数据
> local：表示从本地加载，如果没有则是从 hdfs 加载
> inpath：表示加载数据的路径
> overwrite：表示覆盖表中已有数据，没有则是追加
> into table：追加数据
> table_name：具体的表名
> partition：分区信息

## insert 方式

> 基本插入

语法：`insert into table_name values (value1, value2..)`

> 查询插入

语法：`insert into table_name select field from_table where params`

> 覆盖插入

语法：`insert overwrite table_name`

> 多模式查询（根据多张表查询结果）

语法：`from table_name insert overwrite|into table table_name [partition] select fields [where params]`

其中：`insert overwrite|into table_name [partition] select fields [where params]` 可以多次书写。

注意点：`from table_name` 在最前面，后面的 select 是不需要带 `from table_name` 的。

## 创建并插入

> 方式一：从其他表中查询数据并创建

语法：`create table if not exists table_name as select fields from table_name`

> 方式二：通过 location 指定从哪加载数据并创建

语法：`create table if not exists table_name(column column_type) location '/data/path'`


## import 方式

***注意：此操作必须先用 export 导出后，再将数据导入***

语法：`import table table_name [partition(partition_name=partition_value)] from '/file/path'`

> 导入到一个已存在的表中（空表无结构）

```
hive (default)> import table test from '/export';
Copying data from hdfs://hadoop02:9000/export/data
Copying file: hdfs://hadoop02:9000/export/data/dept.txt
Loading data to table default.test
OK
Time taken: 0.436 seconds
```

> 导入到一个已存在的表中（空表有结构）

```
hive (default)> import table people from '/export';
FAILED: SemanticException [Error 10120]: The existing table is not compatible with the import spec.   Column Schema does not match
```

此时会报 schema 不匹配，原因：import 导入的是 export 的数据，此数据是带有元数据的。

> 导入到一个不存在的表中

```
hive (default)> import table people1 from '/export';
Copying data from hdfs://hadoop02:9000/export/data
Copying file: hdfs://hadoop02:9000/export/data/dept.txt
Loading data to table default.people1
OK
Time taken: 0.32 seconds
```

---

# 数据导出

## insert 导出

> 将查询结果导出的本地

语法：`insert overwrite local directory '/local/file/path' select fields from table_name`

```
hive (default)> insert overwrite local directory '/opt/module/hive/tmp_data/dept' select * from dept;
Query ID = root_20191231145413_67fa5115-2833-40fb-91bd-911b288dea6f
Total jobs = 1
Launching Job 1 out of 1
Number of reduce tasks is set to 0 since there's no reduce operator
Starting Job = job_1577759600542_0001, Tracking URL = http://hadoop03:8088/proxy/application_1577759600542_0001/
Kill Command = /opt/module/hadoop-2.7.2/bin/hadoop job  -kill job_1577759600542_0001
Hadoop job information for Stage-1: number of mappers: 1; number of reducers: 0
2019-12-31 14:54:51,880 Stage-1 map = 0%,  reduce = 0%
2019-12-31 14:55:19,855 Stage-1 map = 100%,  reduce = 0%, Cumulative CPU 9.6 sec
MapReduce Total cumulative CPU time: 9 seconds 600 msec
Ended Job = job_1577759600542_0001
Copying data to local directory /opt/module/hive/tmp_data/dept
Copying data to local directory /opt/module/hive/tmp_data/dept
MapReduce Jobs Launched: 
Stage-Stage-1: Map: 1   Cumulative CPU: 9.6 sec   HDFS Read: 2961 HDFS Write: 69 SUCCESS
Total MapReduce CPU Time Spent: 9 seconds 600 msec
OK
dept.deptno	dept.dname	dept.loc
Time taken: 68.597 seconds
```

```
[root@hadoop02 tmp_data]# cd dept/
[root@hadoop02 dept]# ll
总用量 4
-rw-r--r-- 1 root root 69 12月 31 14:55 000000_0
```

```
[root@hadoop02 dept]# cat 000000_0 
10ACCOUNTING1700
20RESEARCH1800
30SALES1900
40OPERATIONS1700
```

可以看到此时导出的数据是没有格式的

> 将查询结果格式化后导出到本地

语法：`insert overwrite local directory '/local/file/path' ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t' select * from table_name`

```
hive (default)> insert overwrite local directory '/opt/module/hive/tmp_data/dept' ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t' select * from dept;
Query ID = root_20191231150022_adc3b27c-0666-40f8-9577-9689dec1e4ba
Total jobs = 1
Launching Job 1 out of 1
Number of reduce tasks is set to 0 since there's no reduce operator
Starting Job = job_1577759600542_0002, Tracking URL = http://hadoop03:8088/proxy/application_1577759600542_0002/
Kill Command = /opt/module/hadoop-2.7.2/bin/hadoop job  -kill job_1577759600542_0002
Hadoop job information for Stage-1: number of mappers: 1; number of reducers: 0
2019-12-31 15:00:28,451 Stage-1 map = 0%,  reduce = 0%
2019-12-31 15:00:46,034 Stage-1 map = 100%,  reduce = 0%, Cumulative CPU 3.3 sec
MapReduce Total cumulative CPU time: 3 seconds 300 msec
Ended Job = job_1577759600542_0002
Copying data to local directory /opt/module/hive/tmp_data/dept
Copying data to local directory /opt/module/hive/tmp_data/dept
MapReduce Jobs Launched: 
Stage-Stage-1: Map: 1   Cumulative CPU: 3.3 sec   HDFS Read: 3144 HDFS Write: 69 SUCCESS
Total MapReduce CPU Time Spent: 3 seconds 300 msec
OK
dept.deptno	dept.dnam
```

```
[root@hadoop02 tmp_data]# cat dept/000000_0 
10	ACCOUNTING	1700
20	RESEARCH	1800
30	SALES	1900
40	OPERATIONS	1700
```

> 将查询结果打出到 HDFS

语法：`insert overwrite directory '/hdfs/file/path' ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t' select * from table_name`

```
hive (default)> insert overwrite directory '/dept' ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t' select * from dept;
Query ID = root_20191231150613_00f90db5-63a6-4f64-be64-615787dfa988
Total jobs = 3
Launching Job 1 out of 3
Number of reduce tasks is set to 0 since there's no reduce operator
Starting Job = job_1577759600542_0003, Tracking URL = http://hadoop03:8088/proxy/application_1577759600542_0003/
Kill Command = /opt/module/hadoop-2.7.2/bin/hadoop job  -kill job_1577759600542_0003
Hadoop job information for Stage-1: number of mappers: 1; number of reducers: 0
2019-12-31 15:06:23,590 Stage-1 map = 0%,  reduce = 0%
2019-12-31 15:06:27,736 Stage-1 map = 100%,  reduce = 0%, Cumulative CPU 1.21 sec
MapReduce Total cumulative CPU time: 1 seconds 210 msec
Ended Job = job_1577759600542_0003
Stage-3 is selected by condition resolver.
Stage-2 is filtered out by condition resolver.
Stage-4 is filtered out by condition resolver.
Moving data to: hdfs://hadoop02:9000/dept/.hive-staging_hive_2019-12-31_15-06-13_809_519050561514978017-1/-ext-10000
Moving data to: /dept
MapReduce Jobs Launched: 
Stage-Stage-1: Map: 1   Cumulative CPU: 1.21 sec   HDFS Read: 3080 HDFS Write: 69 SUCCESS
Total MapReduce CPU Time Spent: 1 seconds 210 msec
OK
dept.deptno	dept.dname	dept.loc
Time taken: 15.096 seconds
```

![insert 方式导出](/images/hive/db/insert-to-hdfs.png)

## 其他方式导出

> hadoop 命令导出

语法：`hadoop fs -get /hdfs/file/path /hdfs|local/file/path ` 

> Hive shell 导出

语法： `bin/hive -e 'select * from table_name;' > /local/file/path`

> export 导出到 hdfs

语法：`hive (default)> export table table_name to /hdfs/file/path`

```
hive (default)> export table dept to '/export';
Copying data from file:/tmp/root/de98917c-077b-4ec5-a8f5-479a75cc3c12/hive_2019-12-31_15-14-28_860_4397941394477204195-1/-local-10000/_metadata
Copying file: file:/tmp/root/de98917c-077b-4ec5-a8f5-479a75cc3c12/hive_2019-12-31_15-14-28_860_4397941394477204195-1/-local-10000/_metadata
Copying data from hdfs://hadoop02:9000/user/hive/warehouse/dept
Copying file: hdfs://hadoop02:9000/user/hive/warehouse/dept/dept.txt
OK
Time taken: 0.376 seconds
```

![export](/images/hive/db/export.png)

---

# 清空表

语法： `truncate table table_name`

***此操作只能删除管理表，不能删除外部表中的数据***