---
title: Hive(五) <BR/> DDL 3 分区表、修改表
date: 2019-12-30 17:33:55
updated: 2019-12-30 17:33:55
categories:
    [hive, ddl]
tags:
    [hive, ddl]
---

除了管理表、内部表外，常用的表还有分区表。分区表实际上是一个 HDFS 文件夹，文件夹下存放的是对应分区的数据

<!-- more -->

# 分区表

分区表实际上就是对应一个 HDFS 文件系统上的独立的文件夹，该文件夹下是该分区的所有数据。
Hive 中的分区就是分目录，把一个大的数据集根据业务需要，分隔成小的数据集。
在查询的时候，通过 WHERE 字句中的表达式选择查询所需要的指定分区，这样的查询效率会提高很多。

## 基本操作

> 创建一个表，指定分区

```sql
create table dept_partition(
    deptno int, dname string, loc string
)
partitioned by (month string)
row format delimited fields terminated by '\t';
```

> 加载数据查看问题

```
hive (default)> load data local inpath '/opt/module/hive/tmp_data/dept.txt' into table dept_partition;
FAILED: SemanticException [Error 10062]: Need to specify partition columns because the destination table is partitioned
```

可以看到，数据加载失败了。原因是我们创建表的时候，设置了根据 `month` 分区，而在加载数据的时候没有指定分区列造成的。

> 按照分区加载数据

```
hive (default)> load data local inpath '/opt/module/hive/tmp_data/dept.txt' into table dept_partition partition(month='2019-07');
Loading data to table default.dept_partition partition (month=2019-07)
Partition default.dept_partition{month=2019-07} stats: [numFiles=1, numRows=0, totalSize=71, rawDataSize=0]
OK
Time taken: 0.464 seconds
```

![分区加载数据](/images/hive/db/partition.png)

> 再次加载两个分区

![分区加载数据](/images/hive/db/partition-2.png)

> 查询数据

```
hive (default)> select * from dept_partition;
OK
dept_partition.deptno	dept_partition.dname	dept_partition.loc	dept_partition.month
10	ACCOUNTING	1700	2019-07
20	RESEARCH	1800	2019-07
30	SALES	1900	2019-07
40	OPERATIONS	1700	2019-07
10	ACCOUNTING	1700	2019-08
20	RESEARCH	1800	2019-08
30	SALES	1900	2019-08
40	OPERATIONS	1700	2019-08
10	ACCOUNTING	1700	2019-09
20	RESEARCH	1800	2019-09
30	SALES	1900	2019-09
40	OPERATIONS	1700	2019-09
Time taken: 0.226 seconds, Fetched: 12 row(s)
```

可以看到，三个分区的数据都能查到，且把分区参数当做了一列 `dept_partition.month`

> 利用 where 字句查询某分区的数据

```
hive (default)> select * from dept_partition where month='2019-08';
OK
dept_partition.deptno	dept_partition.dname	dept_partition.loc	dept_partition.month
10	ACCOUNTING	1700	2019-08
20	RESEARCH	1800	2019-08
30	SALES	1900	2019-08
40	OPERATIONS	1700	2019-08
Time taken: 1.148 seconds, Fetched: 4 row(s)
```

> 查看元数据

![分区查询](/images/hive/db/partition-3.png)
![分区查询](/images/hive/db/partition-4.png)

## 增加分区

```sql
hive (default)> alter table dept_partition add partition(month='2019-10');
OK
Time taken: 0.202 seconds
```

> 同时添加多个分区

```sql
alter table dept_partition add partition(month='2019-11') partition(month='2019-12');
```

## 删除分区

```sql
hive (default)> alter table dept_partition drop partition(month='2019-12');
Dropped the partition month=2019-12
OK
Time taken: 0.376 seconds
```

> 同时删除多个分区

```sql
hive (default)> alter table dept_partition drop partition(month='2019-11'), partition(month='2019-10');
Dropped the partition month=2019-10
Dropped the partition month=2019-11
OK
Time taken: 0.371 seconds
```

***注意：添加多个分区时，分区与分区之间用 空格 隔开；删除多个分区时，分区与分区之间用 英文逗号 隔开！！***


## 查看分区表有多少分区

```
hive (default)> show partitions dept_partition;
OK
partition
month=2019-07
month=2019-08
month=2019-09
Time taken: 0.055 seconds, Fetched: 3 row(s)
```

## 查看分区结构

```conf
hive (default)> desc formatted dept_partition;
OK
col_name	data_type	comment
# col_name            	data_type           	comment             
	 	 
deptno              	int                 	                    
dname               	string              	                    
loc                 	string              	                    
	 	 
# Partition Information	 	 
# col_name            	data_type           	comment             
	 	 
month               	string       # 此处即是分区的结构       	                    
	 	 
# Detailed Table Information	 	 
Database:           	default             	 
# 省略           
Time taken: 0.062 seconds, Fetched: 34 row(s)
```


## 分区表的注意事项

### 创建二级分区表

```sql
create table dept_partition2(
    deptno int, dname string, loc string
)
partitioned by (month string, day string)
row format delimited fields terminated by '\t';
```

与一级分区的区别是 `partitioned by` 中定义两个字段。

### 加载数据到二级分区

```
hive (default)> load data local inpath '/opt/module/hive/tmp_data/dept.txt' into table dept_partition2 partition(month='2019-10', day='30');
Loading data to table default.dept_partition2 partition (month=2019-10, day=30)
Partition default.dept_partition2{month=2019-10, day=30} stats: [numFiles=1, numRows=0, totalSize=71, rawDataSize=0]
OK
Time taken: 0.358 seconds
```

### 查看分区数据

```
hive (default)> select * from dept_partition2 where month='2019-10' and day='30';
OK
dept_partition2.deptno	dept_partition2.dname	dept_partition2.loc	dept_partition2.month	dept_partition2.day
10	ACCOUNTING	1700	2019-10	30
20	RESEARCH	1800	2019-10	30
30	SALES	1900	2019-10	30
40	OPERATIONS	1700	2019-10	30
Time taken: 0.104 seconds, Fetched: 4 row(s)
```

![多级分区](/images/hive/db/high-partition.png)


## 分区表与数据产生关联的三种方式

### 上传数据后修复

适用场景：在 HDFS 上存在分区目录，且目录中存在文件，但是元数据中没有对应的分区信息。

以 `dept_partition` 为例，此时这个分区表只有三个分区

![分区](/images/hive/db/partition-5.png)

1. 创建一个 `2019-11` 文件夹，上传 dept.txt 数据。

```
[root@hadoop02 tmp_data]# hadoop fs -mkdir -p /user/hive/warehouse/dept_partition/month=2019-10
[root@hadoop02 tmp_data]# hadoop fs -put dept.txt /user/hive/warehouse/dept_partition/month=2019-10
```

2. 查询 `month=2019-10` 分区的数据。

```
hive (default)> select * from dept_partition where month='2019-10';
OK
dept_partition.deptno	dept_partition.dname	dept_partition.loc	dept_partition.month
Time taken: 0.134 seconds
```

3. 此时，执行分区修复

```
hive (default)> msck repair table dept_partition;
OK
Partitions not in metastore:	dept_partition:month=2019-10
Repair: Added partition to metastore dept_partition:month=2019-10
Time taken: 0.201 seconds, Fetched: 2 row(s)
```

4. 再次查询分区
```
hive (default)> select * from dept_partition where month='2019-10';
OK
dept_partition.deptno	dept_partition.dname	dept_partition.loc	dept_partition.month
10	ACCOUNTING	1700	2019-10
20	RESEARCH	1800	2019-10
30	SALES	1900	2019-10
40	OPERATIONS	1700	2019-10
Time taken: 0.088 seconds, Fetched: 4 row(s)
```

### 修改表分区

只需要将上述第三步修改为： `alter table dept_partition where month='2019-10'`

### load 数据

将第一种方式的前三步合并为一步： `load data local inpath '/file/path' into table table_name partition(partition_name=partition_value)`

---

# 修改表

## 表重命名

语法：`alter table table_name rename to new_table_name;`

***此操作会修改元数据和 HDFS 的文件夹名称***

```
hive (default)> alter table test rename to rename_table;
OK
Time taken: 0.185 seconds
```
```
hive (default)> show tables;
OK
tab_name
dept
dept_partition
dept_partition2
emp
people
rename_table
Time taken: 0.016 seconds, Fetched: 6 row(s)
```

## 增加/修改/替换列信息

### 修改列

语法：`alter table table_name CHANGE [column] col_old_name col_new_name column_type [COMMENT col_coment] [FIRST | AFTER column_name]`

```
hive (default)> alter table test CHANGE column name sex int;
OK
Time taken: 0.082 seconds

hive (default)> desc test;
OK
col_name	data_type	comment
id                  	int                 	                    
sex                 	int              	                    
Time taken: 0.054 seconds, Fetched: 2 row(s)
```

### 增加、替换列信息

语法：`alter table table_name ADD | REPLACE COLUMNS (col_name data_type [COMMENT col_comment], ...)`

```
hive (default)> alter table test add columns (name string);
OK
Time taken: 0.132 seconds
hive (default)> select * from test;
OK
test.id	test.name
1	NULL
2	NULL
3	NULL
4	NULL
5	NULL
Time taken: 0.288 seconds, Fetched: 5 row(s)
```

***注意：ADD 表示增加一个字段，字段位置在所有列的后面（partition列的前面）***
***REPLACE 是替换表中的 所有字段！不能只替换一个字段***

```
hive (default)> alter table test replace columns (name string);
OK
Time taken: 0.083 seconds
hive (default)> desc test;
OK
col_name	data_type	comment
name                	string              	                    
Time taken: 0.051 seconds, Fetched: 1 row(s)
```

## 删除表

```
hive (default)> drop table test;
OK
Time taken: 0.206 seconds
```