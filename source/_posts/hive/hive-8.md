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

## 向表中装载数据

### load 方式

语法： `load data [local] inpath '' [overwrite] into table table_name [partition(partition_name=partition_value)]`

> load data：表示加载数据
> local：表示从本地加载，如果没有则是从 hdfs 加载
> inpath：表示加载数据的路径
> overwrite：表示覆盖表中已有数据，没有则是追加
> into table：追加数据
> table_name：具体的表名
> partition：分区信息

### insert 方式

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

### 创建并插入

> 方式一：从其他表中查询数据并创建

语法：`create table if not exists table_name as select fields from table_name`

> 方式二：通过 location 指定从哪加载数据并创建

语法：`create table if not exists table_name(column column_type) location '/data/path'`


### import 方式

***注意：此操作必须先用 export 导出后，再将数据导入***

语法：`import table table_name [partition(partition_name=partition_value)] from '/file/path'`


# 数据导出