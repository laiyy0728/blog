---
title: Spring Cloud 微服务（29） --- Spring Cloud Config(三) <BR> git 版 coofnig 配置、使用数据库实现配置中心
date: 2019-03-05 11:28:06
updated: 2019-03-05 11:28:06
categories: spring-cloud
tags:  [SpringCloud, CloudConfig]
---

除了使用 git 作为配置文件的管理中心外，也可以使用关系型数据库、非关系型数据库实现配置中心，以及配置中心的扩展。包括：客户端自动刷新、客户端回退、安全认证、客户端高可用、服务端高可用等。

<!-- more -->

---

# 服务端 git 配置详解

git 的版 config 有多种配置：
- uri 占位符
- 模式匹配
- 多残酷
- 路径搜索占位符

***源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-config/spring-cloud-placeholder***

公共依赖
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

## git 中 uri 占位符

Spring Cloud Config Server 支持占位符的使用，支持 {application}、{profile}、{label}，这样的话就可以在配置 uri 的时候，通过占位符使用应用名称来区分应用对应的仓库进行使用。

*** config server ***
```yml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://gitee.com/laiyy0728/{application}        # {application} 是匹配符，匹配项目名称
          search-paths: config-simple
  application:
    name: spring-cloud-placeholder-server
server:
  port: 9090
```

*** config client ***

bootstrap.yml
```yml
spring:
  cloud:
    config:
      label: master
      uri: http://localhost:9090
      name: config-repo
      profile: dev
```

使用 `{application}` 时，需要注意，在 config client 中配置的 `name`，既是 config 管理中心的 git 名称，又是需要匹配的配置文件名称。即：远程的 config git 管理中心地址为：https://gitee.com/laiyy0728/config-repo ，在仓库中 `config-simple` 文件夹下，必须有一个 `config-simple.yml` 配置文件。否则 config client 会找不到配置文件。

## 模式匹配、多存储库

***config server***
```yml
spring:
  profiles:
    active: native # 本地配置仓库，在测试本地配置仓库之前，需要注释掉这一行
  cloud:
    config:
      server:
        git:
          uri: https://gitee.com/laiyy0728/config-repo
          search-paths: config-simple
          repos:
            simple: https://gitee.com/laiyy0728/simple
            special:
              pattern: special*/dev*,*special*/dev*
              uri: https://gitee.com/laiyy0728/special
        native:
          search-locations:  C:/Users/laiyy/AppData/Local/Temp/config-simple # 本地配置仓库路径
  application:
    name: spring-cloud-placeholder-server
server:
  port: 9090
```


## 路径搜索占位符
```yml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://gitee.com/laiyy0728/config-repo
          search-paths: config-* # 匹配以 config 开头的文件夹
  application:
    name: spring-cloud-placeholder-server
server:
  port: 9090
```


---

# 关系型数据库实现配置中心

架构图：
![config server mysql](/images/spring-cloud/config/config-db.png)

公共依赖
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

## mysql config server

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-server</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-jdbc</artifactId>
    </dependency>

    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
    </dependency>
</dependencies>
```

```yml
server:
  port: 9090
spring:
  application:
    name: spring-cloud-customer-repo-mysql
  cloud:
    config:
      server:
        jdbc:
          sql: SELECT `KEY`, `VALUE` FROM PROPERTIES WHERE application = ? NAD profile = ? AND label = ?
      label: master
  profiles:
    active: jdbc
  datasource:
    url: jdbc:mysql:///springcloud?useUnicode=true&charsetEncoding=UTF-8
    username: root
    password: 123456
    driver-class-name: com.mysql.cj.jdbc.Driver
logging:
  level:
    org.springframework.jdbc.core: debug
    org.springframework.jdbc.core.StatementCreatorUtils: Trace
```

其余配置、config client 与之前一致即可。

## 验证
启动 config server、config client，可以看到，config server 打印日志如下：
```
...

Executing prepared SQL query
Executing prepared SQL statement [SELECT `KEY`, `VALUE` FROM PROPERTIES WHERE application = ? AND profile = ? AND lable = ?]
Setting SQL statement parameter value: column index 1, parameter value [config-simple], value class [java.lang.String], SQL type unknown
Setting SQL statement parameter value: column index 2, parameter value [dev], value class [java.lang.String], SQL type unknown
Setting SQL statement parameter value: column index 3, parameter value [master], value class [java.lang.String], SQL type unknown
Executing prepared SQL query
Executing prepared SQL statement [SELECT `KEY`, `VALUE` FROM PROPERTIES WHERE application = ? AND profile = ? AND lable = ?]
Setting SQL statement parameter value: column index 1, parameter value [config-simple], value class [java.lang.String], SQL type unknown
Setting SQL statement parameter value: column index 2, parameter value [default], value class [java.lang.String], SQL type unknown
Setting SQL statement parameter value: column index 3, parameter value [master], value class [java.lang.String], SQL type unknown
Executing prepared SQL query
Executing prepared SQL statement [SELECT `KEY`, `VALUE` FROM PROPERTIES WHERE application = ? AND profile = ? AND lable = ?]
Setting SQL statement parameter value: column index 1, parameter value [application], value class [java.lang.String], SQL type unknown
Setting SQL statement parameter value: column index 2, parameter value [dev], value class [java.lang.String], SQL type unknown
Setting SQL statement parameter value: column index 3, parameter value [master], value class [java.lang.String], SQL type unknown
Executing prepared SQL query
Executing prepared SQL statement [SELECT `KEY`, `VALUE` FROM PROPERTIES WHERE application = ? AND profile = ? AND lable = ?]
Setting SQL statement parameter value: column index 1, parameter value [application], value class [java.lang.String], SQL type unknown
Setting SQL statement parameter value: column index 2, parameter value [default], value class [java.lang.String], SQL type unknown
Setting SQL statement parameter value: column index 3, parameter value [master], value class [java.lang.String], SQL type unknown

...
```

访问 http://localhost:9091/get-config-info ，返回数据如下：
![spring cloud config mysql](/images/spring-cloud/config/config-mysql-repo.png)

---

# 非关系数据库实现配置中心

以 mongodb 为例，需要 spring cloud config server mongodb 依赖，github 地址：https://github.com/spring-cloud-incubator/spring-cloud-config-server-mongodb

## config server monngodb

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-server</artifactId>
    </dependency>

    <!-- mongogdb 在 spring cloud config server 的依赖，这个依赖是快照依赖，需要指定 spring 的仓库 -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-server-mongodb</artifactId>
        <version>0.0.3.BUILD-SNAPSHOT</version>
    </dependency>
</dependencies>
```

```yml
server:
  port: 9090
spring:
  application:
    name: spring-cloud-customer-repo-mongodb
  data:
    mongodb:
      uri: mongodb://192.168.67.133/springcloud # mongo 数据库地址
```

```java
@SpringBootApplication
@EnableMongoConfigServer  // 一定注意，不能写为 EnableConfigServer，一定要是 MongooConfigServer
public class SpringCloudCustomerRepoMongodbApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudCustomerRepoMongodbApplication.class, args);
    }

}
```

## mongo 测试数据

collection 名称：springcloud

数据：
```json
{
	"label": "master",
	"profile": "prod",
	"source": {
		"com": {
			"laiyy": {
				"gitee": {
					"config": "I am the mongdb configuration file from dev environment. I will edit."
				}
			}
		}
	}
}
```

## config client

```yml
spring:
  cloud:
    config:
      label: master
      uri: http://localhost:9090
      name: springcloud # 这里指定的是 collection name
      profile: prod
```

## 验证

***config client 与之前的一致***

访问 http://localhost:9091/get-config-info ，返回值为：
![mongo result](/images/spring-cloud/config/mongo-result.png)