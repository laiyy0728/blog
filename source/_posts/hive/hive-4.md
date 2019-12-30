---
title: Hive(四) <BR/> Hive 数据类型、DDL
date: 2019-12-30 09:45:25
updated: 2019-12-30 09:45:25
categories:
    [hive]
tags:
    [hive]
---

Hive 中也有和 Java 类似的基本数据类型、复杂数据类型，也可以进行类型转换。
在之前的操作中，使用的一直都是 default 数据库，如果要新建数据库该如何操作？

<!-- more -->

---

# Hive 的数据类型

## 基本数据类型

| Hive 数据类型 | Java 数据类型 | 长度 | 例子 |
| :-: | :-: | :-: | :-: |
| TINYINT | byte | 1 byte 有符号整数 | 20 |
| SMALLINT | short | 2 byte 有符号整数 | 20 |
| ***INT*** | int | 4 byte 有符号整数 | 20 |
| ***BIGINT*** | long | 8 byte 有符号整数 | 20 |
| BOOLEAN | boolean | 布尔类型 | TRUE、FALSE |
| FLOAT | float | 单精度浮点数 | 3.14 |
| ***DOUBLE*** | double | 双精度浮点数 | 3.14 |
| ***STRING*** | string | 字符类型，可指定字符集，可使用单引号或双引号 | `'test'`、`"test"` |
| TIMESTAMP | | 时间类型 | |
| BINARY | | 字节数组 | |

Hive 是 String 类型，相当于数据库的 varchar 类型。该类型是一个可变字符串，理论上可以存储 2GB 的字符数。

## 集合数据类型

| 数据类型 | 描述 | 语法 |
| :-: | :-: | :-: |
| STRUCT | 结构体，类似于Java Bean，可以通过 a.b 访问 | struct() |
| MAP | 键值对，使用数组标识符可以访问数据 | map() |
| ARRAY | 一组具有相同类型和名称的变量的集合 | Array() |

ARRAY 和 Map 与 Java 中的 Array、Map 类型，STRUCT 与 c 语言中的 struct 类型，封装了一个命名字段集合，复杂数据类型允许任意层次嵌套。

### 示例

***注意：Hive 每次只能解析一行数据，如果是有结构，且美化过的 JSON，Hive 解析不了。***

如下数据，对应 Hive 而言是没有格式的，解析不了。
```json
{
    "name": "zhangsan"
}
```

使用 [测试数据](/file/hive/people.txt)，测试 Hive 的复杂数据类型



测试数据解析：
***laiyy,liyl_nixy,xiao lai:18_xiao yang:19,zheng zhou_henan***
对应为：
```json
{
    "name": "laiyy",
    "friends": ["liyl", "nixy"],
    "children": {
        "xiao lai":18,
        "xiao yang":19
    },
    "address": {
        "street": "zheng zhou",
        "city": "henan"
    }
}
```

> 在 Hive 上创建一个 people 表

```sql
create table people(
    name string,
    friends array<string>,  -- 创建一个 friends 字段，类型为 array，存储 string 
    children map<string, int>, -- 创建一个 children 字段，类型为 map，key 为 string 类型， value 为 int 类型
    address struct<street:string, city:string> -- 创建一个 address 字段，类型为 struct，其中 street 字段为 string 类型，city 字段为 string 类型
) 
row format delimited fields terminated by ',' -- 列分隔符
collection items terminated by '_' -- 数据分隔符，分隔 MAP、STRUCT、ARRAY
map keys terminated by ':' -- MAP 中的 KV 分隔符
lines terminated by '\n' -- 以 \n 分隔每一行
```

HQL 解释为：以 `\n` 分隔每一行，读取一行数据，以 `,` 分隔每个字段的值；如果值中有 `_`，则为数组类型；如果值中有 `:`，则为 map，`:` 分隔 map 的 KV


> 加载数据并查询测试

```
hive (default)> load data local inpath '/opt/module/hive/tmp_data/people.txt' into table people;
Loading data to table default.people
Table default.people stats: [numFiles=1, totalSize=116]
OK
Time taken: 0.899 seconds


hive (default)> select * from people;
OK
people.name	people.friends	people.children	people.address
laiyy	["liyl","nixy"]	{"xiao lai":18,"xiao yang":19}	{"street":"zheng zhou","city":"henan"}
lierg	["cuihua","goudan"]	{"zhang mazi":17,"zhao si":20}	{"street":"kai feng","city":"henan"}
Time taken: 0.293 seconds, Fetched: 2 row(s)
```

由此可见，复杂结构体的测试也通过了。

> 测试单字段查询

```
hive (default)> select friends from people;
OK
friends
["liyl","nixy"]
["cuihua","goudan"]
Time taken: 0.078 seconds, Fetched: 2 row(s)
```

> 查询数组中的某个下标的数据

```
hive (default)> select friends[1] from people;
OK
_c0
nixy
goudan
Time taken: 0.07 seconds, Fetched: 2 row(s)
```

> 查询 map 中的某个 key

```
hive (default)> select children['xiao lai'] from people;
OK
_c0
18
NULL
Time taken: 0.073 seconds, Fetched: 2 row(s)
```

> 查询结构体的某个字段

```
hive (default)> select address.city from people;
OK
city
henan
henan
Time taken: 0.054 seconds, Fetched: 2 row(s)
```

## 类型转换

Hive 的原子数据类型是可以进行 `隐式转换` 的，类似于 Java 的类型转换。如某表达式用 INT 类型，TINYINT 会自动转换为 INT 联系，但是 Hive 不会反向转化（某表达式用 TINYINT，INT 不会自动转为 TINYINT，会返回错误，除非使用强制类型转换 CAST）

### 隐式类型转换规则

> 任何整数类型都可以隐式转换为一个范围更广的类型
> 所有整数类型、FLOAT、STRING(数值) 都行都可以隐式转换为 DOUBLE
> TINYINT、SMALLINT、INT 都可以转换为 FLOAT
> BOOLEAN 不可以转换为其他任何类型

### CAST 强制转换

如 CAST('1' AS INT)，可以将字符串 1 转换为整数 1；如果强制类型转换失败，如 CAST('X' AS INT)，表达式会返回 NULL。


### 测试

```
hive (default)> select CAST('1' AS INT);
OK
_c0
1
Time taken: 0.086 seconds, Fetched: 1 row(s)
```
```
hive (default)> select CAST('abc' AS INT);
OK
_c0
NULL
Time taken: 0.072 seconds, Fetched: 1 row(s)
```

---

# DDL 数据定义

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