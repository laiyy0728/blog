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
> CURRENT ROW：当前行
> n PRECEDING：往前 n 行数据
> n FOLLOWING：往后 n 行数据
> UNBOUNDED：起点
> UNBOUNDED PRECEDING：从前面的起点
> UNBOUNDED FOLLOWING：到后面的起点
> LAG(col, n)：往前第 n 行数据
> LEAD(col, n)：往后第 n 行数据
> NTILE(n)：把有序分区中的行分发到指定数据的组中，各个组有编号，编号从 1 开始；对于每一行，NTILE 返回慈航所属组的编号；n 必须是 int 类型

## 测试

根据 [测试数据](/file/hive/over.txt)，实现以下需求

> 查询在 2017 年 4 月份购买过的顾客及总人数
> 查询顾客的购买明细及月购买总额
> 上述常见要将 cost 按照日期进行累加
> 查询顾客上次的购买时间
> 查询前 20% 时间的订单信息

### 创建表

```sql
hive (default)> create table business(
              > name string,
              > orderdate string,
              > cost int
              > ) ROW FORMAT DELIMITED FIELDS TERMINATED BY ',';
OK
Time taken: 0.072 seconds
```

### 需求 1

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