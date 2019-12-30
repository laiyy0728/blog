---
title: Hive(五) <BR/> DDL
date: 2019-12-30 11:33:02
updated: 2019-12-30 11:33:02
categories:
    [hive, ddl]
tags:
    [hive, ddl]
---


DDL 语句：数据库模型定义语言，用于定义数据库、表的结构。
DDL 语句不仅可以操作数据库，也可以操作数据表。

<!-- more -->


# DDL 库操作

## 创建数据库

注意一定要加上 `if not exists`，避免报错。

```sql
create database if not exists db_hive ;
```
![创建数据库](/images/hive/db/create_db.png)

## 创建数据库，并指定 HDFS 的存储路径

```sql
create database if not exists db_hive2 location '/db_hive2.db';
```

![创建数据库并指定存储路径](/images/hive/db/create_db_2.png)

## 查询数据库

```
hive (default)> show databases;
OK
database_name
db_hive
db_hive2
default
Time taken: 0.013 seconds, Fetched: 3 row(s)
```

> 模糊查询库
```
hive (default)> show databases like 'db_hive*';
OK
database_name
db_hive
db_hive2
Time taken: 0.01 seconds, Fetched: 2 row(s)
```

## 查看数据库详情

```
hive (default)> desc database db_hive;
OK
db_name	comment	location	owner_name	owner_type	parameters
db_hive		hdfs://hadoop02:9000/user/hive/warehouse/db_hive.db	root	USER	
Time taken: 0.016 seconds, Fetched: 1 row(s
```

## 修改数据库

可以使用 ALTER DATABASE 没了为某个数据库设置键值对属性(dbpeoperties)，来描述这个数据库的属性信息。
***数据库的其他元数据信息都是不可更改的，包括数据吗名和数据库所在目录位置。***

```
hive (default)> alter database db_hive set dbproperties('createtime'='20191230');
OK
Time taken: 0.019 seconds
```

> extended 可以查看附加值
```
hive (default)> desc database extended db_hive;
OK
db_name	comment	location	owner_name	owner_type	parameters
db_hive		hdfs://hadoop02:9000/user/hive/warehouse/db_hive.db	root	USER	{createtime=20191230}
Time taken: 0.013 seconds, Fetched: 1 row(s
```

## 删除数据库

```
hive (default)> drop database is not exists db_hive;
OK
Time taken: 0.075 seconds
```

## 强制删除数据库

如果数据库中有数据，可以使用 `CASCADE` 命令强制删除。

```
hive (default)> drop database is not exists db_hive2 cascade;
OK
Time taken: 0.033 seconds
```

# DDL 表操作

## 建表语法

```
CREATE [EXTERNAL] TABLE [IF NOT EXISTS] table_name
[(col_name data_type [COMMENT col_comment], ...)]
[COMMENT table_comment]
[PARTITIONED BY (col_name data_type [COMMENT col_comment], ...)]
[CLUSTERED BY (col_name, col_name, ...)
[SORTED BY (col_name [ASC|DESC], ...)] INTO num_buckets BUCKETS]
[ROW FORMAT row_format]
[STORED AS file_format]
[LOCATION hdfs_path]
```

### 建表语法解释

> CREATE TABLE

创建一个指定名字的表，如果表名称已存在，则会出现异常；可以通过 `IF NOT EXISTS` 来规避。

> EXTERNAL

可以让用户创建一个 `外部表`，在建表的同时指定一个指向实际数据的路径。
***Hive 创建内部表时，会将数据移动到数据仓库指向的路径；若创建外部表，仅记录数据所在的路径，不对数据的位置做任何更改。***
***在删除表的时候，内部表的元数据和数据会被一起删除，而外部表只删除元数据。***

> COMMENT

为表和列添加注释

> PARTITIONED BY

创建分区表

> CLUSTERED BY

创建分桶表，按照列进行分区

> SORTED BY

排序规则，不常用，按照列进行分桶

> ROW FORMAT

行格式化规则
包括：
`DELIMITED [FIELDS TERMINATED BY char] [COLLECTION ITEMS TERMINATED BY char] [MAP KEYS TERMINATED BY char] [LINES TERMINATED BY char]`

`SERDE serde_name [WITH SERDEPROPERTIES](peoperty_name=property_value, ...)`

在建表时可以自定义 serDe（Serialize、Deserialize 的简称，目的是用于序列化、反序列化），或者使用自带的 serDe。
如果没有指定 ROW FORMAT 或 ROW FORMAT DELIMITED，将会使用自带的 serDe。
在建表时，还需要为表指定列，在指定表的列的同事也会指定自定义的 serDe，Hive 通过 serDe 确定表的具体的列的数据

> STORED AS

指定存储文件类型。常用类型为： SEQUENCEFILE（二进制序列文件）、TEXTFILE（文本）、RCFILE（列式存储格式文件）。

如果是纯文本数据，可以使用 `STORED AS TEXTFILE`。如果数据需要压缩，可以使用 `STORED AS SEQUENCEFILE`

> LOCATION

指定表再 HDFS 上的存储位置

> LIKE

允许用户复制现有的表结构，但是不复制数据


### 查看表信息

以之前的 `peopel` 表为例。

```
hive (default)> show create table people;
OK
createtab_stmt
CREATE TABLE `people`(
  `name` string, 
  `friends` array<string>, 
  `children` map<string,int>, 
  `address` struct<street:string,city:string>)
ROW FORMAT DELIMITED 
  FIELDS TERMINATED BY ',' 
  COLLECTION ITEMS TERMINATED BY '_' 
  MAP KEYS TERMINATED BY ':' 
  LINES TERMINATED BY '\n' 
STORED AS INPUTFORMAT 
  'org.apache.hadoop.mapred.TextInputFormat' 
OUTPUTFORMAT 
  'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION
  'hdfs://hadoop02:9000/user/hive/warehouse/people'
TBLPROPERTIES (
  'COLUMN_STATS_ACCURATE'='true', 
  'numFiles'='1', 
  'totalSize'='116', 
  'transient_lastDdlTime'='1577674200')
Time taken: 0.132 seconds, Fetched: 21 row(s)
```

可以看到，除了建表时手动写的一些语句外，还增加了 `STORED AS`、`LOCATIONM`、`TBLPROPERTIES` 等。