---
title: Hive(三) <BR/> JDBC 访问、常用命令
date: 2019-12-26 16:30:50
updated: 2019-12-26 16:30:50
categories:
    [hive]
tags:
    [hive]
---


Hive 的元数据可以存储在 MySQL 中，解决了 derby 只能单客户端连接的问题。Hive 也可以开启类似 JDBC 的连接查询方式。

<!-- more -->

# HiveJDBC 访问

hiveserver2：可以提供一个类似于 JDBC 的连接方式，连接成功后，可以在程序中使用 SQL 进行操作（不是 HQL）。

> 修改 hive-site.xml

在 hive-site.xml 中，增加配置 hiveserver2 连接的 用户名、密码。
***注意：此用户必须拥有 HDFS 中 /tmp/hive 目录的读写权限***
***此用户是系统用户，并不是随意指定的***
```xml
<property>
    <name>hive.server2.thrift.client.user</name>
    <value>root</value>
</property>
<property>
    <name>hive.server2.thrift.client.password</name>
    <value>123456</value>
</property>
```

创建一个 `test.txt`，将 test.txt 存储在 test 表中（`create table test(id int)`）

```
hive> load data local inpath '/opt/module/hive/tmp_data/test.txt' into table test;
Loading data to table default.test
Table default.test stats: [numFiles=1, totalSize=11]
OK
Time taken: 1.355 seconds

hive> select * from test;
OK
1
2
3
4
5
NULL
Time taken: 0.233 seconds, Fetched: 6 row(s)
```

> 启动 hiveserver2：`# bin/hiveserver2`，是个阻塞进程
> 启动 beeline
```
[root@hadoop02 hive]# bin/beeline 
Beeline version 1.2.1 by Apache Hive
beeline> 
```

> 连接 hiveserver2

***如果没有配置 hiveserver2 的用户名、密码，则在进行下面的连接时，会提示连接被拒绝***

***如果在 hadoop 的 `vore-site.xml` 文件中没有如下配置，也会提示连接被拒绝***

```xml
<!-- hadoop 集群的 core-site.xml -->
<property>
    <name>hadoop.http.staticuser.user</name>
    <value>root</value>
</property>
```

> 成功连接

语法： `!connect jdbc:hive2://host:port`
host 为 hive 所在服务器，port 为 10000.

```
beeline> !connect jdbc:hive2://hadoop02:10000
Connecting to jdbc:hive2://hadoop02:10000
Enter username for jdbc:hive2://hadoop02:10000: root
Enter password for jdbc:hive2://hadoop02:10000: ******
Connected to: Apache Hive (version 1.2.1)
Driver: Hive JDBC (version 1.2.1)
Transaction isolation: TRANSACTION_REPEATABLE_READ
```

> 测试

```
0: jdbc:hive2://hadoop02:10000> show tables;
+-----------+--+
| tab_name  |
+-----------+--+
| test      |
+-----------+--+
1 row selected (0.096 seconds)

0: jdbc:hive2://hadoop02:10000> select * from test;
+----------+--+
| test.id  |
+----------+--+
| 1        |
| 2        |
| 3        |
| 4        |
| 5        |
+----------+--+
```

---

# 常用命令

```
[root@hadoop02 hive]# bin/hive -help
usage: hive
 -d,--define <key=value>          Variable subsitution to apply to hive
                                  commands. e.g. -d A=B or --define A=B
    --database <databasename>     Specify the database to use
 -e <quoted-query-string>         SQL from command line
 -f <filename>                    SQL from files
 -H,--help                        Print help information
    --hiveconf <property=value>   Use value for given property
    --hivevar <key=value>         Variable subsitution to apply to hive
                                  commands. e.g. --hivevar A=B
 -i <filename>                    Initialization SQL file
 -S,--silent                      Silent mode in interactive shell
 -v,--verbose                     Verbose mode (echo executed SQL to the
                                  console)
```

## 常用命令

### 命令行执行 hql

执行完后自动退出

```
[root@hadoop02 hive]# bin/hive -e "select * from test;"

Logging initialized using configuration in jar:file:/opt/module/hive/lib/hive-common-1.2.1.jar!/hive-log4j.properties
OK
1
2
3
4
5
Time taken: 1.208 seconds, Fetched:5 row(s)
[root@hadoop02 hive]#
```

### 执行文件内的 hql

创建一个文件： hql.txt，文件内容为
```sql
select * from test;
```

> 执行

```
[root@hadoop02 hive]# bin/hive -f tmp_data/hql.txt 

Logging initialized using configuration in jar:file:/opt/module/hive/lib/hive-common-1.2.1.jar!/hive-log4j.properties
OK
1
2
3
4
5
Time taken: 1.155 seconds, Fetched: 5 row(s)
```

## hive 命令

### 查看 hdfs 

```
hive> dfs -ls /;
Found 2 items
drwx-wx-wx   - root supergroup          0 2019-12-23 15:17 /tmp
drwxr-xr-x   - root supergroup          0 2019-12-23 15:18 /user
```

### 查看本地文件系统

```
hive> ! ls /opt/module;
hadoop-2.7.2
hive
jdk1.8.0_144
zookeeper-3.4.10
```

### 查看历史命令

进入当前用户的根目录，查看 `.hivehistory` 文件

```
[root@hadoop02 ~]# cat .hivehistory 
show databaes;
show databases;
use default;
create table stu(id int, name string) row format delimited fields terminated by '\t';
quit;
show tables;
create table test(id int);
show tables;
load data local inpath '/opt/module/hive/tmp_data/test.txt' into table test;
select * from test;
quit;
```

--- 

# 常用配置

## Hive 数据仓库位置

> 默认数据仓库的最原始位置在 HDFS 上，路径为： `/user/hive/warehouse/`
> 在仓库目录下，没有对默认的数据库 default 创建文件夹。如果某张表属于 default 库，则会直接在数据仓库目录下创建一个文件夹
> 修改 default 数据仓库的原始位置（hive-site.xml）

```xml
<property>
    <name>hive.metastore.warehouse.dir</name>
    <value>/user/hive/warehouse</value>
    <description>默认仓库存储路径</description>
</property>
```

## 查询后信息显示的配置

> 在 hive-site 中增加配置，显示当前数据库，以及查询表的头信息配置

```xml
<property>
    <name>hive.cli.print.current.db</name>
    <value>false</value>
    <description>是否打印当前数据库</description>
</property>
<property>
    <name>hive.cli.print.header</name>
    <value>false</value>
    <description>是否打印头信息</description>
</property>
```

测试：
```
hive (default)> select * from test;
OK
test.id
1
2
3
4
5
```


可以看到库名：hive(default)。头信息：test.id

## hive 运行日志

> Hive 默认的日志存放在 /tmp/[当前用户]/hive.log 下

```
[root@hadoop02 hive]# ll /tmp/root/
总用量 3708
-rw-r--r--  1 root root 2292655 12月 27 16:01 hive.log
-rw-r--r--. 1 root root   25667 12月 23 15:18 hive.log.2019-12-23
-rw-r--r--  1 root root 1470990 12月 26 15:27 hive.log.2019-12-26
```

将 `hive-log4j.properties.template` 复制一份为 `hive-log4j.properties`，修改 log 存储位置

```conf
#hive.log.dir=${java.io.tmpdir}/${user.name}
hive.log.dir=/opt/logs/hive
```

修改后重启 hive 即可。

## 参数配置方式

> 查看 mapreduce task 数量

```
hive (default)> set mapred.reduce.tasks;
mapred.reduce.tasks=-1
```

> 设置 mapreduce task 数量

```
hive (default)> set mapred.reduce.tasks=10;
hive (default)> set mapred.reduce.tasks;
mapred.reduce.tasks=10
```

> 另外一种设置方式

```
[root@hadoop02 hive]# bin/hive -hiveconf mapred.reduce.tasks=10;

Logging initialized using configuration in file:/opt/module/hive/conf/hive-log4j.properties
hive (default)> set mapred.reduce.tasks;
mapred.reduce.tasks=10
```