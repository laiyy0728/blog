---
title: Hive(九) <BR/> 数据查询（1）
date: 2019-12-31 15:30:55
updated: 2019-12-31 15:30:55
categories:
    [hive]
tags:
    [hive]
---

Hive 的查询与 sql 基本一致，都有基本查询、别名、limit、like、分组、join 等语法。

查询语句语法：https://cwiki.apache.org/confluence/display/Hive/LanguageManual+Select

<!-- more -->

```sql
SELECT [ALL | DISTINCT] select_expr, select_expr, ...
  FROM table_reference
  [WHERE where_condition]
  [GROUP BY col_list]
  [ORDER BY col_list]
  [CLUSTER BY col_list
    | [DISTRIBUTE BY col_list] [SORT BY col_list]
  ]
 [LIMIT [offset,] rows]
```

# 基本查询

## 全表和特定列查询

> 全表查询

```sql
select * from table_name;
```

> 特定列查询

```sql
select col_name from table_name;
```

> 注意

1. HQL 语句大小写不敏感
2. HQL 可以写在一行或多行
3. 关键字不能被缩写，也不能分行
4. 各子句一般要分行写
5. 使用缩进提高语句的可读性

## 列别名查询

> 重命名一个列、便于计算
> 紧跟列名，也可以在列名与别名之间加 `AS`

```SQL
select col_name new_name from table_name;
select col_name AS new_name from table_name;
```

## 算数运算符

| 运算符 | 描述 |
| :-: | :-: |
| A + B | A 和 B 相加 |
| A - B | A 和 B 相减 |
| A * B | A 和 B 相乘 |
| A / B | A 除以 B |
| A % B | A 对 B 取余 |
| A & B | A 和 B 按位取 与 |
| A | B | A 和 B 按位取 或 |
| A ^ b | A 和 B 按位取 异或 |
| ~A | A 按位取反 |

```sql
select col_name + 1 from table_name;
```

## 常用内置函数

> 求总行数

```sql
select count(*) from table_name;
```

> 求最大、最小值

```sql
select max(col_name) from table_name;
select min(col_name) from table_name;
```

> 求总和、平均值

```sql
select sum(col_name) from table_name;
select avg(col_name) from table_name;
```

## limit

```sql
select * from table_name limit num;
```

---

# Where 子句

> 使用 where 子句，将不满足条件的行过滤掉
> where 子句紧随 from 子句

```sql
select * from table_name where condition;
```

## 比较运算符

> 谓词操作符，可以用于 where、join...on、having 子句中

| 操作符 | 支持的数据类型 | 描述 |
| :-: | :-: | :-: |
| A = B | 基本数据类型 | 如果 A 等于 B，返回 TRUE，否则返回 FALSE |
| A <=> B | 基本数据类型 | 如果 A 和 B 都为 NULL，则返回 TRUE，其他的和（=）操作符一致；如果任意一项为 NULL，则结果为 NULL |
| A <> B，A != B | 基本数据类型 | 如果 A 或 B 为 NULL，则返回 NULL；如果 A 不等于 B，返回 TRUE，否则返回 FALSE |
| A < B | 基本数据类型 | A 或者 B 为 NULL，则返回 NULL；如果 A 小于 B，则返回 TRUE，反之返回 FALSE |
| A <= B | 基本数据类型 | A 或者 B 为 NULL，则返回 NULL；如果 A 小于等于 B，则返回 TRUE，否则返回 FALSE |
| A > B | 基本数据类型 | A 或者 B 为 NULL，则返回 NULL；如果 A 大于 B，则返回 TRUE，反之返回 FALSE |
| A >= B | 基本数据类型 | A 或者 B 为 NULL，则返回 NULL；如果 A 大于等于 B，则返回 TRUE，否则返回 FALSE |
| A [NOT] BETWEEN B AND C | 基本数据类型 | 如果 A， B 或者 C 任一为 NULL，则结果为 NULL。如果 A 的值大于等于 B 而且小于或等于 C，则结果为 TRUE，反之为 FALSE。如果使用 NOT 关键字则可达到相反的效果。|
| A IS NULL | 所有数据类型 | 如果 A 等于 NULL，则返回 TRUE，反之返回 FALSE |
| A IS NOT NULL | 所有数据类型 | 如果 A 等于 NULL，则返回 TRUE，反之返回 FALSE |
| IN(数值 1, 数值 2) | 所有数据类型 | 使用 IN 运算显示列表中的值 |
| A [NOT] LIKE B | STRING 类型 | B 是一个 SQL 下的简单正则表达式，如果 A 与其匹配的话，则返回 TRUE；反之返回 FALSE。 B 的表达式说明如下：‘x%’表示 A必须以字母‘x’开头，‘%x’表示 A 必须以字母’ x’结尾，而‘%x%’表示 A 包含有字母’ x’ ,可以位于开头，结尾或者字符串中间。如果使用 NOT 关键字则可达到相反的效果。 |
| A RLIKE B, A REGEXP B | STRING 类型 | B 是一个正则表达式，如果 A 与其匹配，则返回 TRUE；反之返回FALSE。匹配使用的是 JDK 中的正则表达式接口实现的，因为正则也依据其中的规则。例如，正则表达式必须和整个字符串 A 相匹配，而不是只需与其字符串匹配。 |

## Like、RLike

> 使用 LIKE 运算选择类似的值
> 选择条件可以包含字符、数字。（`%` 代表 0 个或多个字符；`_` 代表一个字符）
> RLIKE 子句是 Hive 中的功能扩展，可以通过 Java 正则表达式来指定匹配条件

## 逻辑运算符

| 操作符 | 含义 |
| :-: | :-: |
| AND | 逻辑与 |
| OR | 逻辑或 |
| NOT | 逻辑或 |

---

# 分组函数

## GROUP BY

GROUP BY 语句通常会和聚合函数一起使用，按照一个或多个队列结果进行分组展示，然后对每个组执行聚合操作。

```
hive (default)> select deptno, avg(sal) avg_sal from emp group by deptno;
Query ID = root_20200106101803_35d15cb7-040e-42ea-8345-0ea0975951f8

...

Total MapReduce CPU Time Spent: 4 seconds 310 msec
OK
deptno	avg_sal
10	2916.6666666666665
20	2175.0
30	1566.6666666666667
Time taken: 25.732 seconds, Fetched: 3 row(s)
```

## Having 子句

### Having 与 where 不同点

> where 针对表中的列发挥作用；having 针对查询结果中的列发生作用，筛选数据
> where 后面不能写分组函数；having 后面可以使用分组函数
> having 只用于 group by 分组统计语句

```
hive (default)> select deptno, avg(sal) avg_sal from emp group by deptno having avg_sal > 2000;
Query ID = root_20200106102306_7985f86c-fa99-4348-9daf-cac6bdd2ced8

...

Total MapReduce CPU Time Spent: 4 seconds 180 msec
OK
deptno	avg_sal
10	2916.6666666666665
20	2175.0
Time taken: 25.657 seconds, Fetched: 2 row(s)
```

---

# Join

## 等值 join

Hive 直冲通常的 SQL JOIN，但是只支持等值连接，不支持非等值连接。
***Hive 只支持 on A=B；不支持 on A!=B***

```
hive (default)> select e.deptno, e.ename from emp e join dept d on e.deptno = d.deptno;
Query ID = root_20200106102604_eaba33e7-9e47-4a91-b8db-401f0e258c4b

...

Total MapReduce CPU Time Spent: 2 seconds 50 msec
OK
e.deptno	e.ename
20	SMITH

...

10	MILLER
Time taken: 18.614 seconds, Fetched: 14 row(s)
```

## 表的别名

```sql
select * from table_name table_alias_name;
```

> 好处

1. 使用别名可以优化查询
2. 使用表名前缀可以提高执行效率

## 内连接

只有进行连接的两个表都存在连接条件相匹配的数据才会被保留下来。

```SQL
hive (default)> select e.empno, e.ename, d.deptno from emp e join dept d on e.deptno = d.deptno;
```

## 左外、右外、满外连接

> 左外连接

JOIN 操作符左边表中符合 WHERE 子句的记录会被记录下来。

```sql
hive (default)> select e.empno, e.ename, d.deptno from emp e left join dept d on e.deptno = d.deptno;
```

> 右外连接

JOIN 操作符右边表符合 WHERE 子句的记录会被记录下来。

```SQL
hive (default)> select e.empno, e.ename, d.deptno from emp e right dept d on e.deptno = d.deptno;
```

> 满外链接

会返回所有表中符合 WHERE 语句条件的所有记录。如果任意表的指定字段没有符合条件的值，则会用 NULL 替代。

```SQL
hive (default)> select e.empno, e.ename, d.deptno from emp e full dept d on e.deptno = d.deptno;
```

## 多表 join

***注意：连接 N 个表，至少需要 N-1 个连接条件***

大多数情况下，Hive 会对每对 JOIN 对象启动一个 MapReduce 任务


## 笛卡尔集

> 产生条件

1. 省略连接条件
2. 连接条件无效
3. 所有表中的所有行互相连接

```
hive (default)> select empno,dname from emp,dept;
Warning: Map Join MAPJOIN[7][bigTable=emp] in task 'Stage-3:MAPRED' is a cross product

...

Total MapReduce CPU Time Spent: 1 seconds 210 msec
OK
empno	dname
7369	ACCOUNTING

...

7934	OPERATIONS
Time taken: 14.668 seconds, Fetched: 56 row(s)
```

## 连接谓词不支持 OR

```
hive (default)> select e.empno, e.ename, d.deptno from emp e join dept d on e.deptno = d.deptno or e.ename = d.ename;
FAILED: SemanticException [Error 10019]: Line 1:60 OR not supported in JOIN currently 'ename'
```

---

# 排序

## 全局排序

Order By：全局排序，一个 Reducer

> 使用 Order by 子句排序（ASC：升序，默认；DESC：降序
> Order by 子句在 SELECT 语句的末尾
> 具体操作与 SQL 类型

## 内部排序

Sort by：对每个 Reducer 内部有进行排序，对全局结果集来说不是排序。

1. 设置 Reducer 数量 `set mapreduce.job.reduces=3`
2. 查询数据 `hive (default)> select * from emp sort by empno desc;`
3. 将查询结果导入到文件中（按照部门编号降序）`hive (default)> insert overwrite local directory '/opt/module/datas/hive/sortby-result' select * from emp sort by deptno desc;`
4. 查看导出的结果

```
[root@hadoop02 ~]# ll /opt/module/datas/hive/sortby-result
总用量 12
-rw-r--r-- 1 root root 288 1月   6 10:56 000000_0
-rw-r--r-- 1 root root 282 1月   6 10:56 000001_0
-rw-r--r-- 1 root root  91 1月   6 10:56 000002_0
[root@hadoop02 ~]# 
```

## 分区排序

Distribute by：类似 MR 中的 partition 进行分区，需要结合 sort by 使用。

***注意：Hive 要求 Distribute by 语句要写在 sort by 语句之前***

对于 Distribute by 进行测试，一定要分配多个 reduce 进行处理，否则无法看到 Distribute by 的效果。

> 测试：先按照部门编号分区，再按照员工编号降序排序

```SQL
hive (default)> insert overwrite local directory '/opt/module/datas/hive/distributeby-result' select * from emp distribute by deptno sort by empno desc;
```

> 执行结果

```
[root@hadoop02 ~]# ll /opt/module/datas/hive/distributeby-result
总用量 12
-rw-r--r-- 1 root root 293 1月   6 11:03 000000_0
-rw-r--r-- 1 root root 139 1月   6 11:03 000001_0
-rw-r--r-- 1 root root 229 1月   6 11:03 000002_0
```

> 可以看到在 000000_0 中，都是部门 id 为 30 的。

```
[root@hadoop02 ~]# cat /opt/module/datas/hive/distributeby-result/000000_0 
7900JAMESCLERK76981981-12-3950.0\N30
7844TURNERSALESMAN76981981-9-81500.00.030
7698BLAKEMANAGER78391981-5-12850.0\N30
7654MARTINSALESMAN76981981-9-281250.01400.030
7521WARDSALESMAN76981981-2-221250.0500.030
7499ALLENSALESMAN76981981-2-201600.0300.030
```

## Cluster by

当 distribute by 和 sort by 字段相同时，可以使用 cluster by 方式。

cluster by 兼具 distribute by 和 sort by 的功能。

只能升序；不能指定排序规则为 ASC、DESC。

> 下面两种写法等价

```sql
select * from emp cluster by deptno;
select * from emp distribute by deptno sort by deptno;
```