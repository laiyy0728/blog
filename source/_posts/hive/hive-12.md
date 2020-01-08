---
title: Hive(十二) <BR/> 常用函数——窗口函数
date: 2020-01-07 16:55:17
updated: 2020-01-07 16:55:17
categories:
    [hive]
tags:
    [hive]
---

`select deptno, count(*) from emp group by mgr`。这条语句如果直接执行是会报错的，此时就可以用窗口函数来解决。

<!-- more -->

# 窗口函数

OVER：指定分析函数工作的数据窗口大小，这个数据窗口大小可能会随着行的变化而变化。

OVER 函数可能出现的连接属性：
> 用在 over 函数内的
>> CURRENT ROW：当前行
>> n PRECEDING：往前 n 行数据
>> n FOLLOWING：往后 n 行数据
>> UNBOUNDED：起点
>> UNBOUNDED PRECEDING：从前面的起点
>> UNBOUNDED FOLLOWING：到后面的起点

> 用在 over 函数外
>> LAG(col, n, default_value)：往前第 n 行数据
>> LEAD(col, n, default_value)：往后第 n 行数据
>> NTILE(n)：把有序分区中的行分发到指定数据的组中，各个组有编号，编号从 1 开始；对于每一行，NTILE 返回慈航所属组的编号；n 必须是 int 类型


**根据 [测试数据](/file/hive/over.txt)，实现以下需求**

## 需求

> [需求 1](#%e9%9c%80%e6%b1%82-1)：查询在 2017 年 4 月份购买过的顾客及总人数
> [需求 2](#%e9%9c%80%e6%b1%82-2)：查询顾客的购买明细及总购买总额
> [需求 3](#%e9%9c%80%e6%b1%82-3)：基于上述场景，将 cost 按照日期进行累加
> [需求 4](#%e9%9c%80%e6%b1%82-4)：查询顾客的购买明细及每个顾客的购买总额
> [需求 5](#%e9%9c%80%e6%b1%82-5)：查询顾客购买明细，并将 cost 按累加金额（与需求 3 不同之处在于需求 3 是累加总额，此需求是累加每个顾客的总额）
> [需求 6](#%e9%9c%80%e6%b1%82-6)查询顾客上次的购买时间
> [需求 7](#%e9%9c%80%e6%b1%82-7)查询前 20% 时间的订单信息

## 创建表

```sql
hive (default)> create table business(
              > name string,
              > orderdate string,
              > cost int
              > ) ROW FORMAT DELIMITED FIELDS TERMINATED BY ',';
OK
Time taken: 0.072 seconds
```

## 需求 1

```sql
select name, count(*) from business where substring(orderdate, 1, 7) = '2017-04' group by name;
```

用此语句的查询结果为：
```
name	_c1
jack	1
mart	4
Time taken: 36.572 seconds, Fetched: 2 row(s)
```

这个结果是查询出来的是 2017 年 4 月份购买过的顾客，可这个顾客购买的次数，并不是总人数。所以这个方案是不行的。

> 使用窗口函数进行查询

1. 在 count(*) 后加 over()
2. count(*) 与 over() 使用空格隔开即可

```sql
select name, count(*) over() from business where substring(orderdate, 1, 7) = '2017-04' group by name;
```


此时的查询结果是正确的。
```
name	count_window_0
mart	2
jack	2
Time taken: 37.903 seconds, Fetched: 2 row(s)
```

1. over() 函数必须跟在 `聚合函数` 的后面
2. over() 函数调用时也叫做 `开窗`，意义在于对整个数据集开了个窗口
3. over() 开的窗口仅仅是给 count(*) 这个聚合函数用的
4. 对每个数据都开了一个窗口

> 流程

去掉 `count(*) over()` 后，运行剩余语句，结果集为
```
name
mart
jack
```

在加上 `count(*) over()` 后，由于 over() 函数没有传参，代表针对所有结果集生效。此时结果集供有 2 条数据，则会开两个窗口。
即为 mart 开了一个窗，窗口大小是全局的，即为 2，则此时对开窗后的数据进行 count，则 count 为 2；同理 jack 的 count 也为 2。
开窗计算结束后，返回的 count 就是 2，则运行结果就是现在展示出来的样子。

> group by 的流程
![group by](/images/hive/db/group-by.png)

> over 流程
![over](/images/hive/db/over.png)

## 需求 2

```SQL
hive (default)> select *, sum(cost) over() from business;
Query ID = root_20200108145947_5ea91675-0dd0-4b53-bbb0-4d1324bc52d9

...

business.name	business.orderdate	business.cost	sum_window_0
mart	2017-04-13	94	661

...

jack	2017-01-01	10	661
Time taken: 37.215 seconds, Fetched: 14 row(s)

```

## 需求 3

可以看到，每个月的总额是当月销售额加上之前月份的销售额的总和。

```sql
hive (default)> select orderdate, cost, sum(cost) over(order by orderdate)
              >  from business;
Query ID = root_20200108150530_45152897-77dd-4167-af1e-13c945b31b31

...

orderdate	cost	sum_window_0
2017-01-01	10	10
2017-01-02	15	25
2017-01-04	29	54

... 

2017-04-13	94	569
2017-05-10	12	581
2017-06-12	80	661
Time taken: 17.268 seconds, Fetched: 14 row(s)
```

## 需求 4

思路：按照顾客进行分组。

通常情况下，分组用 group by 即可，但需要注意的是，窗口函数是不能用 group by 的。可以使用类似的 distribute by 来进行操作。

```sql
hive (default)> select *, sum(cost) over(distribute by name) from business;

...

OK
business.name	business.orderdate	business.cost	sum_window_0
jack	2017-01-05	46	176
jack	2017-01-08	55	176
jack	2017-01-01	10	176
jack	2017-04-06	42	176
jack	2017-02-03	23	176
mart	2017-04-13	94	299
mart	2017-04-11	75	299
mart	2017-04-09	68	299
mart	2017-04-08	62	299
neil	2017-05-10	12	92
neil	2017-06-12	80	92
tony	2017-01-04	29	94
tony	2017-01-02	15	94
tony	2017-01-07	50	94
Time taken: 19.168 seconds, Fetched: 14 row(s)
```


## 需求 5

```SQL
hive (default)> select *, sum(cost) over(distribute by name sort by orderdate) from business;
Query ID = root_20200108153257_3b1c7cdb-ed63-4f8d-ac93-46a090a51c27

...

OK
business.name	business.orderdate	business.cost	sum_window_0
jack	2017-01-01	10	10
jack	2017-01-05	46	56
jack	2017-01-08	55	111
jack	2017-02-03	23	134
jack	2017-04-06	42	176
mart	2017-04-08	62	62
mart	2017-04-09	68	130
mart	2017-04-11	75	205
mart	2017-04-13	94	299
neil	2017-05-10	12	12
neil	2017-06-12	80	92
tony	2017-01-02	15	15
tony	2017-01-04	29	44
tony	2017-01-07	50	94
Time taken: 17.41 seconds, Fetched: 14 row(s)
```

## 需求 6

```
hive (default)> select *, lag(orderdate, 1, '1970-01-01') over(distribute by name sort by orderdate) from business;
Query ID = root_20200108154129_6bc92c4b-4095-46a9-92bb-4e261d0cfcfa

OK
business.name	business.orderdate	business.cost	lag_window_0
jack	2017-01-01	10	1970-01-01
jack	2017-01-05	46	2017-01-01
jack	2017-01-08	55	2017-01-05
jack	2017-02-03	23	2017-01-08
jack	2017-04-06	42	2017-02-03
mart	2017-04-08	62	1970-01-01
mart	2017-04-09	68	2017-04-08
mart	2017-04-11	75	2017-04-09
mart	2017-04-13	94	2017-04-11
neil	2017-05-10	12	1970-01-01
neil	2017-06-12	80	2017-05-10
tony	2017-01-02	15	1970-01-01
tony	2017-01-04	29	2017-01-02
tony	2017-01-07	50	2017-01-04
Time taken: 24.159 seconds, Fetched: 14 row(s)
```

## 需求 7

> ntile(5) over(order by orderdate)：将数组分为 5 个组，按照时间排序
> where ntile_5 = 1：查询第一个组，即前 20%

```sql
hive (default)> select name, orderdate, cost from (
              > select name, orderdate, cost, ntile(5) over(order by orderdate) ntile_5 from business
              > ) t1 where ntile_5 = 1;
Query ID = root_20200108155352_e7cd86d2-1949-4cfc-91a8-18e6fe11a3ec

...

OK
name	orderdate	cost
jack	2017-01-01	10
tony	2017-01-02	15
tony	2017-01-04	29
Time taken: 17.475 seconds, Fetched: 3 row(s)
```

---

# Rank 函数

Rank 排名函数。

> RANK()：排序相同时会重复，总数不变
> DENSE_RANK()：排序相同时会重复，总数减少
> ROW_NUMER()：根据顺序计算

使用 [测试数据](/file/hive/rank.txt) 测试 Rank 函数

```sql
create table score(
name string,
subject string,
score int)
row format delimited fields terminated by "\t";
```

## 同时测试三种 rank 方式

```sql
hive (default)> select *, rank() over(partition by subject order by score desc) rank1,
              > dense_rank() over(partition by subject order by score desc) rank2,
              > row_number() over(partition by subject order by score desc) rank3
              > from score;
Query ID = root_20200108160934_e99e5b37-0a2c-41c8-a545-4394a10f14e4

...

score.name	score.subject	score.score	rank1	rank2	rank3
孙悟空	数学	95	1	1	1
猪八戒	数学	86	2	2	2
婷婷	数学	85	3	3	3
大海	数学	56	4	4	4
猪八戒	英语	84	1	1	1
大海	英语	84	1	1	2
婷婷	英语	78	3	2	3
孙悟空	英语	68	4	3	4
大海	语文	94	1	1	1
孙悟空	语文	87	2	2	2
婷婷	语文	65	3	3	3
猪八戒	语文	64	4	4	4
Time taken: 21.184 seconds, Fetched: 12 row(s)
```