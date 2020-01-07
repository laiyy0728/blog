---
title: Hive(十) <BR/> 常用函数
date: 2020-01-07 11:28:54
updated: 2020-01-07 11:28:54
categories:
    [hive]
tags:
    [hive]
---

Hive 和普通的数据库一样也有一些常用函数。如果：NVL、CASE WHEN、时间转换等。

<!-- more -->

# 常用查询函数

## 空字段赋值 NVL

> NVL：给值为 NULL 的数据赋值，语法为： NVL(str, replace_with)

如果 str 为 NULL，则 NVL 函数返回 replace_with 的值，否则返回 str 的值。
如果两个参数都是 NULL，则返回 NULL。

第二个参数可以传入一个固定的值，也可以传一个表中的列。

### 测试

使用 [员工数据](/file/hive/emp.txt) 作测试。

> 查询员工的 comm 列数据

可以看到 comm 列的数据有 NULL 值存在。

```
hive (default)> select comm from emp;
OK
comm
NULL
300.0
500.0

...

NULL
NULL
NULL
Time taken: 0.052 seconds, Fetched: 28 row(s)
```

> 使用 NVL 查询 comm 列数据，为 NULL 的置为 -1

```
hive (default)> select nvl(comm, -1) from emp;
OK
_c0
-1.0
300.0

...

-1.0
-1.0
-1.0
Time taken: 0.063 seconds, Fetched: 28 row(s)
```

## 时间类

***注意：时间转换方面，只支持 yyyy-MM-dd 格式的年月日，和 HH:mm:ss 格式的时分秒。不支持其他时间分隔符。***
***如果时间格式不是 yyyy-MM-dd HH:mm:ss 格式，可以使用 regexp_replace(date, from, to) 来替换时间。***
**如： `regexp_replace('2020/01/07', '/', '-')`**

> date_format 格式化时间

```
hive (default)> select date_format('2020-01-07', 'yyyy-MM-dd');
OK
_c0
2020-01-07
Time taken: 0.091 seconds, Fetched: 1 row(s)
```

> date_add 时间跟天数相加

```
hive (default)> select date_add('2020-01-07', 5);
OK
_c0
2020-01-12
Time taken: 0.062 seconds, Fetched: 1 row(s)
```

> date_sub 时间跟天数相减

```
hive (default)> select date_sub('2020-01-07', 5);
OK
_c0
2020-01-02
Time taken: 0.055 seconds, Fetched: 1 row(s)
```

> datediff 两个时间相减

```
hive (default)> select datediff('2020-01-07', '2020-01-01');
OK
_c0
6
Time taken: 0.24 seconds, Fetched: 1 row(s)
```

## CASE WHEN

用途：当 case 的字段符合 when 的条件时，输出某个值。

使用 [测试数据](/file/hive/case-when.txt)，创建数据表

```sql
hive (default)> create table emp_sex(name string, dept_id string, sex string) row format delimited fields terminated  by '\t';
OK
Time taken: 0.208 seconds
```

需求：输出不同的 dept_id 下，男女各有多少人。

```sql
select dept_id, 
sum(case sex when '男' then 1 else 0 end) male_count, 
sum(case sex when '女' then 1 else 0 end) female_count 
from emp_sex 
group by dept_id;
```

查询结果：
```
Query ID = root_20200107153632_eb5c0874-e757-4489-b4b1-800eb49293bf

...

OK
dept_id	male_count	female_count
A	2	2
B	2	1
Time taken: 34.46 seconds, Fetched: 2 row(s)
```

## 行列转换

### 行转列

> CONCAT(str A/col, str B/col ...)

返回输入字符串连接后的结果，支持任意个输入字符串。

```
hive (default)> select concat(deptno, '-', dname) from dept;
OK
_c0
10-ACCOUNTING
20-RESEARCH
30-SALES
40-OPERATIONS
Time taken: 0.078 seconds, Fetched: 4 row(s)
```

> CONCAT_WS(separator, str1, str2...)

特殊形式的 CONCAT 函数。第一个参数代表分隔符。分隔符可以是与其余参数一样的字符串，如果分隔符是 NULL，返回值也是 NULL。这个函数会跳过分隔符参数后的的任何 NULL 和空字符串。分隔符将被加载到被连接的字符串之间。

```
hive (default)> select concat_ws('-', 'h', 'e', 'l', 'l', 'o');
OK
_c0
h-e-l-l-o
Time taken: 0.084 seconds, Fetched: 1 row(s)
```

> COLLECT_SET(col)

只接受基本类型数据，主要作用是将某字段的值进行去重、汇总，产生 array 类型字段。

```
hive (default)> select collect_set(loc) from dept;
Query ID = root_20200107160149_1b18e6d7-20ed-4e5f-b153-2837a5441989

...

OK
_c0
[1700,1800,1900]
Time taken: 29.301 seconds, Fetched: 1 row(s
```

**测试：根据 [输入数据](/file/hive/concat.txt)，将星座、血型一样的人归类到一起**

> 数据表

```
hive (default)> create table person_info(
              > name string,
              > constellation string,
              > blood_type string)
              > row format delimited fields terminated by '\t';
OK
Time taken: 0.396 seconds
```

> 查询

```sql
hive (default)> select
              >     concat(constellation, ',', blood_type) constellation_blood_type,
              >     name
              > from person_info;
OK
constellation_blood_type	name
白羊座,A	孙悟空
射手座,B	孙二娘
白羊座,A	猪八戒
白羊座,A	沙悟净
射手座,B	白龙马
白羊座,A	唐三藏
Time taken: 0.049 seconds, Fetched: 6 row(s)
```

> 在上例基础上，进行再次查询


```SQL
hive (default)> select constellation_blood_type, concat_ws('|', collect_set(name))
              >     from (
              >         select
              >             concat(constellation, ',', blood_type) constellation_blood_type,
              >             name
              >         from peoson_info )
              >      t1 
              > group by constellation_blood_type;
Query ID = root_20200107162303_31323702-6403-42da-9cc0-294d4ae05870

...

OK
constellation_blood_type	_c1
射手座,A	白龙马
射手座,B	孙二娘
白羊座,A	孙悟空|沙悟净|唐三藏
白羊座,B	猪八戒
Time taken: 24.091 seconds, Fetched: 4 row(s)
```

### 列转行

> EXPLODE(col)

将 Hive 一列中复杂的 array 或 map 结构拆分成多行。

> LATERAL VIEW（侧写）

用法：LATERAL VIEW udtf(expression) tableAlias as colimnAlias;
解释：用于和 split、explode 等 UDTF(一进多出) 一起使用，它能够将一系列数据拆分成多行数据，再此基础上可以对拆分后的数据进行聚合。

***所需库表***

```sql
hive (default)> create table movie_info(
              >     movie string,
              >     category array<string>
              > )
              > row format delimited fields terminated by '\t'
              > collection items terminated by ',';
OK
Time taken: 0.088 seconds
```

根据 [测试数据](/file/hive/explode.txt)

```
hive (default)> select explode(category) from movie_info;
OK
col
悬疑
科幻
动作
剧情
悬疑
警匪
动作
剧情
心理
战争
动作
灾难
Time taken: 0.042 seconds, Fetched: 12 row(s)
```


> 测试输出电影名称和分类

可以看到测试是不成功的，原因就是 UDTF 不支持
```
hive (default)> select movie,explode(category) from movie_info;
FAILED: SemanticException [Error 10081]: UDTF's are not supported outside the SELECT clause, nor nested in expressions
```

> 修改查询方式

```
hive (default)> select movie, category_name from movie_info lateral view explode(category) table_tmp as category_name;
OK
movie	category_name
《疑犯追踪》	悬疑
《疑犯追踪》	科幻
《疑犯追踪》	动作
《疑犯追踪》	剧情
《Lie to me》	悬疑
《Lie to me》	警匪
《Lie to me》	动作
《Lie to me》	剧情
《Lie to me》	心理
《战狼2》	战争
《战狼2》	动作
《战狼2》	灾难
Time taken: 0.062 seconds, Fetched: 12 row(s)
```