---
title: Hive(六) <BR/> DDL 2
date: 2019-12-30 14:28:26
updated: 2019-12-30 14:28:26
categories:
    [hive, ddl]
tags:
    [hive, ddl]
---

创建表的时候，分为 `管理表` 和 `外部表`。管理表也称为 `内部表`。

<!-- more -->

# 管理表

默认创建的表都是管理表（内部表）。Hive 会控制表中数据的生命周期，Hive 默认情况下会将这些表的数据存储在由配置项 `hive.metastore.warehouse.dir` 所定义的目录的子目录下。
当我们删除一个管理表时，Hive 也会删除这个表中的数据。
管理表不适合和其他工具共享数据。

## 测试

### 创建普通表

```sql
create table if not exists students(
    id int, name string
) 
row format delimited fields terminated by '\t'
stored as textfile
location '/user/hive/warehouse/student2'
```

### 根据查询结果创建表（查询的结果会添加的新创建的表中）

```sql
create table if not exists people2 as select name, friends, children, address from people;
```

```
hive (default)> create table if not exists people2 as select name, friends, children, address from people;
Query ID = root_20191230143721_e7ecd58e-3605-4e64-a4d5-daa161afdc82
Total jobs = 3
Launching Job 1 out of 3
Number of reduce tasks is set to 0 since there's no reduce operator
Starting Job = job_1577671926689_0001, Tracking URL = http://hadoop03:8088/proxy/application_1577671926689_0001/
Kill Command = /opt/module/hadoop-2.7.2/bin/hadoop job  -kill job_1577671926689_0001
Hadoop job information for Stage-1: number of mappers: 1; number of reducers: 0
2019-12-30 14:37:32,362 Stage-1 map = 0%,  reduce = 0%
2019-12-30 14:37:42,730 Stage-1 map = 100%,  reduce = 0%, Cumulative CPU 1.55 sec
MapReduce Total cumulative CPU time: 1 seconds 550 msec
Ended Job = job_1577671926689_0001
Stage-4 is selected by condition resolver.
Stage-3 is filtered out by condition resolver.
Stage-5 is filtered out by condition resolver.
Moving data to: hdfs://hadoop02:9000/user/hive/warehouse/.hive-staging_hive_2019-12-30_14-37-21_922_8266176305010522642-1/-ext-10001
Moving data to: hdfs://hadoop02:9000/user/hive/warehouse/people2
Table default.people2 stats: [numFiles=1, numRows=2, totalSize=116, rawDataSize=114]
MapReduce Jobs Launched: 
Stage-Stage-1: Map: 1   Cumulative CPU: 1.55 sec   HDFS Read: 3867 HDFS Write: 188 SUCCESS
Total MapReduce CPU Time Spent: 1 seconds 550 msec
OK
name	friends	children	address
Time taken: 22.204 seconds
```


```
hive (default)> select * from people2;
OK
people2.name	people2.friends	people2.children	people2.address
iyy	["liyl","nixy"]	{"xiao lai":18,"xiao yang":19}	{"street":"zheng zhou","city":"henan"}
lierg	["cuihua","goudan"]	{"zhang mazi":17,"zhao si":20}	{"street":"kai feng","city":"henan"}
Time taken: 0.052 seconds, Fetched: 2 row(s)
```


---


# 外部表

Hive 并不完全拥有这份数据。当删除表的时候，实际数据不会删除，但描述这个表的元数据会被删除。

## 测试

### 创建部门、员工两张表

```sql
create external table if not exists default.dept(
deptno int,
dname string,
loc int
)
row format delimited fields terminated by '\t';
```


```sql
create external table if not exists default.emp(
empno int,
ename string,
job string,
mgr int,
hiredate string,
sal double,
comm double,
deptno int)
row format delimited fields terminated by '\t';
```

### 导入数据

向两张表中导入 [部门数据](/file/hive/dept.txt) 和 [员工数据](/file/hive/emp.txt)

```
hive (default)> load data local inpath '/opt/module/hive/tmp_data/dept.txt' into table dept;
Loading data to table default.dept
Table default.dept stats: [numFiles=1, totalSize=71]
OK
Time taken: 0.231 seconds
```

```
hive (default)> load data local inpath '/opt/module/hive/tmp_data/emp.txt' into table emp;
Loading data to table default.emp
Table default.emp stats: [numFiles=1, totalSize=669]
OK
Time taken: 0.198 seconds
```

### 删除 dept

```
hive (default)> drop table dept;
OK
Time taken: 0.42 seconds
```

![删除表](/images/hive/db/drop-dept.png)

可以看到，删除表后，数据还在。

### 再次创建 dept 表

再次使用上面的建表语句，把 dept 表建出来，但是 ***不导入任何数据***

使用查询语句查询 dept

```
hive (default)> create external table if not exists default.dept(
              > deptno int,
              > dname string,
              > loc int
              > )
              > row format delimited fields terminated by '\t';
OK
Time taken: 0.043 seconds
```

```
hive (default)> select * from dept;
OK
dept.deptno	dept.dname	dept.loc
10	ACCOUNTING	1700
20	RESEARCH	1800
30	SALES	1900
40	OPERATIONS	1700
Time taken: 0.052 seconds, Fetched: 4 row(s)
```

可以看到，如果数据存在，那么创建表只要能指定到这个 HDFS 文件夹，那么就能查出来数据。


---


# 管理表与外部表的互相转换

## 查询表的类型

```conf
hive (default)> desc formatted test;
OK
col_name	data_type	comment
# col_name            	data_type           	comment             
	 	 
id                  	int                 	                    
	 	 
# Detailed Table Information	 	 
# 省略                 	 
Location:           	hdfs://hadoop02:9000/user/hive/warehouse/test	 
Table Type:         	MANAGED_TABLE     # 此处即为表类型，可以看到是管理表  	 
# 省略      
	 	 
# Storage Information	 	 
SerDe Library:      	org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe	 
# 省略

```

## 修改为外部表

```
hive (default)> alter table test set tblproperties('EXTERNAL'='TRUE');
OK
Time taken: 0.129 seconds
```

## 再次查看表类型

``` conf
hive (default)> desc formatted test;
OK
col_name	data_type	comment
# col_name            	data_type           	comment             
	 	 
id                  	int                 	                    
	 	 
# Detailed Table Information	 	 
Database:           	default             	 
# 省略                 	 
Location:           	hdfs://hadoop02:9000/user/hive/warehouse/test	 
Table Type:         	EXTERNAL_TABLE      	 # 可以看到表类型已经改为了外部表 
# 省略         
	 	 
# Storage Information	 	 
SerDe Library:      	org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe	 
# 省略
```

## 再次改为管理表

```
hive (default)> alter table test set tblproperties('EXTERNAL'='FALSE');
OK
Time taken: 0.129 seconds
```

## 注意点

> EXTERNAL 必须为大写
> 设置是否是外部表只能通过 EXTERNAL 为 TRUE 或 FALSE 来控制
> `'EXTERNAL'='FALSE'`、`'EXTERNAL'='TRUE'`，固定写法，区分大小写


