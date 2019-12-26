---
title: Hive(二) <BR/> 本地文件导入 Hive、MySQL 存储元数据
date: 2019-12-23 10:42:07
updated: 2019-12-23 10:42:07
categories:
    [hive]
tags:
    [hive]
---

除了使用命令创建表、插入数据外，也可以将本地文件数据导入 Hive。

<!-- more -->

# 本地文件导入 Hive

准备一个 [待导入文件](/file/hive/student.txt)，将文件放在 `/opt/module/datas` 文件夹下。

需要注意语法： `load data local inpath [local path] into table [table name]`

```
hive> load data local inpath '/opt/module/datas/student.txt' into table student;
Loading data to table default.student
Table default.student stats: [numFiles=3, numRows=0, totalSize=66, rawDataSize=0]
OK
Time taken: 1.397 seconds
```

> 查看 WebUI

![load data](/images/hive/install/load-data.png)

> 查询

```
hive> select * from student;
OK
1	laiyy
2	laiyy1
NULL	NULL
NULL	NULL
NULL	NULL
NULL	NULL
Time taken: 0.282 seconds, Fetched: 6 row(s)
```

***出现了四条 NULL 记录***

## 解决办法

在建表的时候，就指定分隔符，且导入时，需要按照建表时指定的分隔符来分隔数据。

> 新建一张表，以 \t 分隔

注意语法：`create table [table name] fow format  delimited fields terminated by [terminated type]`

```
hive> create table stu(id int, name string) row format delimited fields terminated by '\t';
OK
Time taken: 0.12 seconds
hive> show tables;
OK
stu
student
Time taken: 0.02 seconds, Fetched: 2 row(s)
```

> 导入数据到 stu 表，查看表数据

```
hive> load data local inpath '/opt/module/datas/student.txt' into table stu;
Loading data to table default.stu
Table default.stu stats: [numFiles=1, totalSize=49]
OK
Time taken: 0.148 seconds


hive> select * from stu;
OK
1001	zhangsan
1002	lisi
1003	wangwu
1004	zhaoliu
Time taken: 0.041 seconds, Fetched: 4 row(s)
```

![insert data tu stu](/images/hive/install/inser-data2.png)

## 将本地文件 put 进表

再创建一个文件 [stu.txt](/file/hive/stu.txt)

```
hadoop fs -put stu.txt /user/hive/warehouse/stu
```

![put tu table](/images/hive/install/put-to-table.png)

> 调用 select * 查看数据

```
hive> select * from stu;
OK
1005	haha
1006	heihei
NULL	NULL
1001	zhangsan
1002	lisi
1003	wangwu
1004	zhaoliu
Time taken: 0.052 seconds, Fetched: 7 row(s)
```

## 将 HDFS 上的数据 put 进表

将 `student.txt` 复制为 `student2.txt` 文件，并上传至 HDFS 根目录下，然后将根目录下的数据 put 到 stu 表中

注意语法：`load data inpath [hdfs path] into table [table name]`

```
[root@hadoop02 datas]# hadoop fs -put student2.txt /


hive> load data inpath '/student2.txt' into table stu;
Loading data to table default.stu
Table default.stu stats: [numFiles=3, totalSize=121]
OK
Time taken: 0.132 seconds


hive> select * from stu;
OK
1005	haha
1006	heihei
NULL	NULL
1001	zhangsan
1002	lisi
1003	wangwu
1004	zhaoliu
1001	zhangsan
1002	lisi
1003	wangwu
1004	zhaoliu
```

> 此时再查看 WebUI，发现根目录下的 /student2.txt 文件消失了

![没有 student2.txt](/images/hive/install/no-student2.png)

其实，student2.txt 文件并没有删除，是 hive 修改了 student2.txt 文件的元数据，从原本的 `/ 目录` 修改为 `/user/hive/warehouse/stu 目录`


---


# 元数据存储在 MySQL

在 hadoop02 上，连接 Hive 后，使用 XShell 再开一个窗口，还是 hadoop02 ，在另外一个窗口没有关闭的情况下，再连接 Hive，会报如下错误

```
Caused by: javax.jdo.JDOFatalDataStoreException: Unable to open a test connection to the given database. JDBC url = jdbc:derby:;databaseName=metastore_db;create=true, username = APP. Terminating connection pool (set lazyInit to true if you expect to start your database after your app). Original Exception: ------
java.sql.SQLException: Failed to start database 'metastore_db' with class loader sun.misc.Launcher$AppClassLoader@214c265e, see the next exception for details.
	at org.apache.derby.impl.jdbc.SQLExceptionFactory40.getSQLException(Unknown Source)
	at org.apache.derby.impl.jdbc.Util.newEmbedSQLException(Unknown Source)
	at org.apache.derby.impl.jdbc.Util.seeNextException(Unknown Source)
	at org.apache.derby.impl.jdbc.EmbedConnection.bootDatabase(Unknown Source)
	at org.apache.derby.impl.jdbc.EmbedConnection.<init>(Unknown Source)
	at org.apache.derby.impl.jdbc.EmbedConnection40.<init>(Unknown Source)
	at org.apache.derby.jdbc.Driver40.getNewEmbedConnection(Unknown Source)
	at org.apache.derby.jdbc.InternalDriver.connect(Unknown Source)
	at org.apache.derby.jdbc.Driver20.connect(Unknown Source)
```

> 错误原因

Hive 默认数据库是 `derby`，是单用户的，当有两个连接以上时就会报错。为了防止这个错误，可以把元数据信息从 derby 中改为存储在 mysql 中。

## 使用 MySQL 存储元数据

***注意：本例假设 MySQL 已经成功安装，且 MySQL 已经配置了 root+密码，所有主机访问***

本例采用MySQL 57 版本，插件包为 [6.0.6 版本](/file/hive/mysql-connector-java-6.0.6.jar)

将插件包拷贝到 `%HIVE_HOME%/lib/` 下。

```
[root@hadoop02 mysql-connector-java-5.1.27]# cp mysql-connector-java-5.1.27-bin.jar /opt/module/hive/lib/
```

> 修改 Hive 配置信息

在 `/opt/module/hive/conf/` 下创建一个 `hive-site.xml` 文件，根据 [官方文档](https://cwiki.apache.org/confluence/display/Hive/AdminManual+Metastore+3.0+Administration#AdminManualMetastore3.0Administration-Option2:ExternalRDBMS)，在 `hive-site.xml` 中添加数据库配置

***注意：数据库要先创建好，表会在启动时自动创建；文件名称必须为 hive-site.xml，否则无效***

```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
    <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <value>jdbc:mysql://hadoop02:3306/hive_meta</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>com.mysql.jdbc.Driver</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionUserName</name>
        <value>root</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionPassword</name>
        <value>123456</value>
    </property>
</configuration>
```

> 启动 hive

由于版本问题导致的 SSL 日志忽略即可。

```
[root@hadoop02 hive]# bin/hive

Logging initialized using configuration in jar:file:/opt/module/hive/lib/hive-common-1.2.1.jar!/hive-log4j.properties
Thu Dec 26 15:18:55 CST 2019 WARN: Establishing SSL connection without server's identity verification is not recommended. According to MySQL 5.5.45+, 5.6.26+ and 5.7.6+ requirements SSL connection must be established by default if explicit option isn't set. For compliance with existing applications not using SSL the verifyServerCertificate property is set to 'false'. You need either to explicitly disable SSL by setting useSSL=false, or set useSSL=true and provide truststore for server certificate verification.
Thu Dec 26 15:18:55 CST 2019 WARN: Establishing SSL connection without server's identity verification is not recommended. According to MySQL 5.5.45+, 5.6.26+ and 5.7.6+ requirements SSL connection must be established by default if explicit option isn't set. For compliance with existing applications not using SSL the verifyServerCertificate property is set to 'false'. You need either to explicitly disable SSL by setting useSSL=false, or set useSSL=true and provide truststore for server certificate verification.
Thu Dec 26 15:18:55 CST 2019 WARN: Establishing SSL connection without server's identity verification is not recommended. According to MySQL 5.5.45+, 5.6.26+ and 5.7.6+ requirements SSL connection must be established by default if explicit option isn't set. For compliance with existing applications not using SSL the verifyServerCertificate property is set to 'false'. You need either to explicitly disable SSL by setting useSSL=false, or set useSSL=true and provide truststore for server certificate verification.
Thu Dec 26 15:18:55 CST 2019 WARN: Establishing SSL connection without server's identity verification is not recommended. According to MySQL 5.5.45+, 5.6.26+ and 5.7.6+ requirements SSL connection must be established by default if explicit option isn't set. For compliance with existing applications not using SSL the verifyServerCertificate property is set to 'false'. You need either to explicitly disable SSL by setting useSSL=false, or set useSSL=true and provide truststore for server certificate verification.
Thu Dec 26 15:18:56 CST 2019 WARN: Establishing SSL connection without server's identity verification is not recommended. According to MySQL 5.5.45+, 5.6.26+ and 5.7.6+ requirements SSL connection must be established by default if explicit option isn't set. For compliance with existing applications not using SSL the verifyServerCertificate property is set to 'false'. You need either to explicitly disable SSL by setting useSSL=false, or set useSSL=true and provide truststore for server certificate verification.
Thu Dec 26 15:18:56 CST 2019 WARN: Establishing SSL connection without server's identity verification is not recommended. According to MySQL 5.5.45+, 5.6.26+ and 5.7.6+ requirements SSL connection must be established by default if explicit option isn't set. For compliance with existing applications not using SSL the verifyServerCertificate property is set to 'false'. You need either to explicitly disable SSL by setting useSSL=false, or set useSSL=true and provide truststore for server certificate verification.
Thu Dec 26 15:18:56 CST 2019 WARN: Establishing SSL connection without server's identity verification is not recommended. According to MySQL 5.5.45+, 5.6.26+ and 5.7.6+ requirements SSL connection must be established by default if explicit option isn't set. For compliance with existing applications not using SSL the verifyServerCertificate property is set to 'false'. You need either to explicitly disable SSL by setting useSSL=false, or set useSSL=true and provide truststore for server certificate verification.
Thu Dec 26 15:18:56 CST 2019 WARN: Establishing SSL connection without server's identity verification is not recommended. According to MySQL 5.5.45+, 5.6.26+ and 5.7.6+ requirements SSL connection must be established by default if explicit option isn't set. For compliance with existing applications not using SSL the verifyServerCertificate property is set to 'false'. You need either to explicitly disable SSL by setting useSSL=false, or set useSSL=true and provide truststore for server certificate verification.


hive> 
```

![启动成功后](/images/hive/install/mysql.png)

> 执行一些测试

查询不到表，是因为 mysql 只是存储 `元数据`，而现在还没有在 mysql 做元数据存储的基础上创建 hive 表，所以查询不到。

```
hive> show tables;
OK
Time taken: 0.016 second
```

创建一个表，再查询

```
hive> create table test(id int);
OK
Time taken: 0.208 seconds
hive> show tables;
OK
test
Time taken: 0.018 seconds, Fetched: 1 row(s)
```

![MySQL metadata](/images/hive/install/mysql-metadata.png)
![MySQl metadata](/images/hive/install/mysql-metadata-1.png)
![MySQl metadata](/images/hive/install/mysql-metadata-2.png)


<!-- ## 安装 MySQL  -->
<!-- 使用 rpm 包安装 mysql，本例版本为 `5.6.24`，如已经安装了 MySQL，可跳过此步骤

> 解压 mysql-libs，并安装 mysql rpm

注意：需要先安装好 `net-tools`、`perl`、`perl-devel` 依赖

```
yum install -y net-tools perl perl-devel
```

> 安装 mysql、启动 mysql

```
#yum install wget

#wget http://repo.mysql.com//mysql57-community-release-el7-7.noarch.rpm

#yum install mysql57-community-release-el7-7.noarch.rpm

#yum install mysql-community-server

# systemctl start mysqld.service
```

> 修改 Mysql 密码

启动后查看 `/var/log/mysqld.log` 文件，查看 mysql 密码，也可能查看不到，查看不到的情况下，修改 `/etc/my.cnf` 文件

```conf
[mysqld]

datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock

symbolic-links=0

log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid

# mysql 登录跳过密码校验
skip-grant-tables=1
```

重启mysql，随便输入密码登录
```
# systemctl restart mysqld.service

[root@hadoop02 etc]# mysql -uroot -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
```

> 修改密码

```
mysql> use mysql
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> update user set authentication_string = password('123456'),password_expired = 'N', password_last_changed = now() where user = 'root';
Query OK, 1 row affected, 1 warning (0.00 sec)

mysql> exit
Bye
```

> 将 `my.cnf` 文件的 `skip-grant-tables=1` 注释掉，重启 mysql， 使用新的登录密码访问。
> 修改允许远程链接

```
mysql> update user set `host` = '%' where `user` = 'root';

mysql> flush privileges;
```

> 重启mysql，使用 Navicat 连接测试

![测试 MySQL](/images/hive/install/mysql-test.png) -->


