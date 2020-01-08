---
title: Hive(十三) <BR/> 窗口函数案例
date: 2020-01-08 16:13:03
updated: 2020-01-08 16:13:03
categories:
    [hive]
tags:
    [hive]
---

对于窗口函数的一些案例 Demo。

<!-- more -->

# 窗口函数

基于 [窗口函数示例](/hive/hive-12.html)，对窗口函数的使用的深化。

## 查询顾客的购买明细及月购买总额

```SQL
hive (default)> select *, sum(cost) over(partition by month(orderdate)) from business;
Query ID = root_20200108161623_4bbe9b57-f59c-4d9c-822b-79b08bbd35ca

...

OK
business.name	business.orderdate	business.cost	sum_window_0
jack	2017-01-01	10	205
jack	2017-01-08	55	205
tony	2017-01-07	50	205
jack	2017-01-05	46	205
tony	2017-01-04	29	205
tony	2017-01-02	15	205
jack	2017-02-03	23	23
mart	2017-04-13	94	341
jack	2017-04-06	42	341
mart	2017-04-11	75	341
mart	2017-04-09	68	341
mart	2017-04-08	62	341
neil	2017-05-10	12	12
neil	2017-06-12	80	80
Time taken: 18.658 seconds, Fetched: 14 row(s)
```

## 按照 name 分组，组内数据累加

> 方式 1

```SQL
hive (default)> select *, sum(cost) over(partition by name order by orderdate) user_cost_sum from business;

...

OK
business.name	business.orderdate	business.cost	user_cost_sum
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
Time taken: 15.966 seconds, Fetched: 14 row(s)
```

> 方式 2

```sql
hive (default)> select *, sum(cost) over(partition by name order by orderdate rows between UNBOUNDED PRECEDING and current row) user_cost_sum from business;

...

OK
business.name	business.orderdate	business.cost	user_cost_sum
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
Time taken: 15.97 seconds, Fetched: 14 row(s)
```

## 按照 name 分组，当前行和前一行数据聚合

```sql
hive (default)> select *, sum(cost) over(partition by name order by orderdate rows between 1 PRECEDING and current row) user_cost_sum from business;
Query ID = root_20200108162136_c51bee16-c128-41fd-815f-706abae6748c

...

OK
business.name	business.orderdate	business.cost	user_cost_sum
jack	2017-01-01	10	10
jack	2017-01-05	46	56
jack	2017-01-08	55	101
jack	2017-02-03	23	78
jack	2017-04-06	42	65
mart	2017-04-08	62	62
mart	2017-04-09	68	130
mart	2017-04-11	75	143
mart	2017-04-13	94	169
neil	2017-05-10	12	12
neil	2017-06-12	80	92
tony	2017-01-02	15	15
tony	2017-01-04	29	44
tony	2017-01-07	50	79
Time taken: 15.016 seconds, Fetched: 14 row(s)
```

## 按照 name 分组，当前行和前一行、后一行数据整合

```sql
hive (default)> select *, sum(cost) over(partition by name order by orderdate rows between 1 PRECEDING and 1 FOLLOWING ) user_cost_sum from business;

...

OK
business.name	business.orderdate	business.cost	user_cost_sum
jack	2017-01-01	10	56
jack	2017-01-05	46	111
jack	2017-01-08	55	124
jack	2017-02-03	23	120
jack	2017-04-06	42	65
mart	2017-04-08	62	130
mart	2017-04-09	68	205
mart	2017-04-11	75	237
mart	2017-04-13	94	169
neil	2017-05-10	12	92
neil	2017-06-12	80	92
tony	2017-01-02	15	44
tony	2017-01-04	29	94
tony	2017-01-07	50	79
Time taken: 14.928 seconds, Fetched: 14 row(s)
```

## 按照 name 分组，当前行和后面所有行聚合

```SQL
hive (default)> select *, sum(cost) over(partition by name order by orderdate rows between CURRENT ROW and UNBOUNDED FOLLOWING ) user_cost_sum from business;
Query ID = root_20200108162420_c5908a2e-c7fa-4bc7-87ad-236a1701d95a

...

business.name	business.orderdate	business.cost	user_cost_sum
jack	2017-01-01	10	176
jack	2017-01-05	46	166
jack	2017-01-08	55	120
jack	2017-02-03	23	65
jack	2017-04-06	42	42
mart	2017-04-08	62	299
mart	2017-04-09	68	237
mart	2017-04-11	75	169
mart	2017-04-13	94	94
neil	2017-05-10	12	92
neil	2017-06-12	80	80
tony	2017-01-02	15	94
tony	2017-01-04	29	79
tony	2017-01-07	50	50
Time taken: 19.561 seconds, Fetched: 14 row(s)
```

---

# 统计每个用户的累计访问次数

使用 [测试数据](/file/hive/action.txt)，完成需求。

```sql
create table action(
    userId string,
    visitDate string,
    visitCount int
)
row format delimited fields terminated by '\t';
```

思路：简化查询，一步步深入。

> 第一步：时间格式化

```SQL
select userId, date_format(regexp_replace(visitDate, '/', '-'), 'yyyy-MM') mn, visitCount from action
```

执行结果：

```
OK
userid	mn	visitcount
u01	2017-01	5
u02	2017-01	6
u03	2017-01	8
u04	2017-01	3
u01	2017-01	6
u01	2017-02	8
u02	2017-01	6
u01	2017-02	4
Time taken: 0.06 seconds, Fetched: 8 row(s)
```

> 第二步：查询每个用户每个月的总和

```sql
select userId, mn, sum(visitCount) xj from (
        select userId, date_format(regexp_replace(visitDate, '/', '-'), 'yyyy-MM') mn, visitCount from action
        ) t1 
    group by userId, mn
```

执行结果：

```
OK
userid	mn	xj
u01	2017-01	11
u01	2017-02	12
u02	2017-01	12
u03	2017-01	8
u04	2017-01	3
Time taken: 16.011 seconds, Fetched: 5 row(s)
```

> 第三步：根据用户累加

```sql
select userId, mn, xj, sum(xj) over(partition by userId order by mn) user_sum from (
    select userId, mn, sum(visitCount) xj from (
        select userId, date_format(regexp_replace(visitDate, '/', '-'), 'yyyy-MM') mn, visitCount from action
        ) t1 
    group by userId, mn
) t2;
```

执行结果

```
OK
userid	mn	xj	user_sum
u01	2017-01	11	11
u01	2017-02	12	23
u02	2017-01	12	12
u03	2017-01	8	8
u04	2017-01	3	3
Time taken: 16.125 seconds, Fetched: 5 row(s)
```