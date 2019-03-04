---
title: Spring Cloud 微服务（27） --- Spring Cloud Config(一) <BR> 配置中心、实例
date: 2019-03-04 09:51:30
updated: 2019-03-04 09:51:30
categories:
    Java
tags:
    [SpringCloud, CloudConfig]
---


Spring Cloud Config 是 Spring Cloud 微服务体系中的配置中心，是微服务中不可或缺的一部分，其能够很好的将程序中配置日益增多的各种功能的开关、参数的配置、服务器的地址等配置修改后实时生效、灰度发布，分环境、分集群管理配置等进行全面的集中化管理，有利于系统的配置管理、维护。

<!-- more -->

# Spring Cloud Config 配置中心

## 配置中心对比

| 对比方面 | 重要性 | SpringCloud Config | Netflix archaius | 携程 Apollo | disconf |
| :-: | :-: | :-: | :-: | :-: | :-: |
| 静态配置管理 | 高 | 基于 file | 无 | 支持 | 支持 |
| 动态配置管理 | 高 | 支持 | 支持 | 支持 | 支持 |
| 统一管理 | 高 | 无，需要 git、数据库等 | 无 | 支持 | 支持 |
| 多维度管理 | 中 | 无，需要 git、数据库等 | 无| 支持 | 支持 |
| 变更管理 | 高 | 无，需要 git、数据库等 | 无 | 无 | 无 |
| 本地配置缓存 | 高 | 无 | 无 | 支持 | 支持 |
| 配置更新策略 | 中 | 无 | 无 | 无 | 无 |
| 配置锁 | 中 | 支持 | 不支持 | 不支持 | 不支持 |
| 配置校验 | 中 | 无 | 无 | 无 | 无 |
| 配置生效时间 | 高 | 重启生效、手动刷新 | 手动刷新失效 | 实时 | 实时 |
| 配置更新推送 | 高 | 需要手动触发 | 需要手动触发 | 支持 | 支持 |
| 配置定时拉取 | 高 | 无 | 无 | 支持 | 配置更新目前依赖事件驱动，client 重启或者 server 推送操作 |
| 用户权限管理 | 中 | 无，需要 git、数据库等 | 无 | 支持 | 支持 |
| 授权、审核、审计 | 中 | 无，需要 git、数据库等 | 无 | 界面直接提供发布历史、回滚按钮 | 操作记录存在数据库中，但是无查询接口 |
| 配置版本管理 | 高 | git | 无 | 支持 | 操作记录存在数据库中，但是无查询接口 |
| 配置合规检测 | 高 | 不支持 | 不支持 | 支持（不完整） | |
| 实例配置监控 | 高 | 需要结合 spring admin | 不支持 | 支持 | 支持，可以查看每个配置再哪台机器上加载 |
| 灰度发布 | 中 | 不支持 | 不支持 | 支持 | 不支持部分更新 |
| 告警通知 | 中 | 不支持 | 不支持 | 支持邮件方式告警 | 支持邮件方式告警 |
| 统计报表 | 中 | 不支持 | 不支持 | 不支持 | 不支持 |
| 依赖关系 | 高 | 不支持 | 不支持 | 不支持 | 不支持 |
| 支持 SpringBoot | 高 | 原生支持 | 低 | 支持 | 与 SpringBoot 无关 |
| 支持 Spring Config | 高 | 原生支持 | 低 | 支持 | 与 SpringBoot 无关 |
| 客户端支持 | 低 | java | java | java、.net | java |
| 业务系统入侵 | 高 | 入侵性弱 | 入侵性弱 | 入侵性弱 | 入侵性弱、支持注解和 xml |
| 单点故障 | 高 | 支持 HA 部署 | 支持 HA 部署 | 支持 HA 部署 | 支持 HA 部署、高可用由 zk 提供 |
| 多数据中心部署 | 高 | 支持 | 支持 | 支持 | 支持 |
| 配置界面 | 中 | 无，需要 git、数据库等 | 无 | 统一界面 | 统一界面 |

## 配置中心具备的功能

- Open API
- 业务无关性
- 配置生效监控
- 一致性 K-V 存储
- 统一配置实时推送
- 配合灰度与更新
- 配置全局恢复、备份、历史
- 高可用集群

![spring cloud 配置中心功能图](/images/spring-cloud/config/config-1.png)

## 配置中心流转

配置中心各流程流转如图：
![Spring cloud 配置中心流转图](/images/spring-cloud/config/config-2.png)


## 配置中心支撑体系

配置中心的支撑体系大致有两类
- 开发管理体系
- 运维管理体系
![Spring 开发、运维体系](/images/spring-cloud/config/config-devops.png)

---

# Spring Cloud Config

## Spring Cloud Config 概述

Spring Cloud Config 是一个集中化、外部配置的分布式系统，由服务端、客户端组成，它不依赖于注册中心，是一个独立的配置中心。Spring Cloud Config 支持多种存储配置信息的形式，目前主要有 jdbc、vault、Navicat、svn、git 等形式，默认为 git。

## git 版工作原理

配置客户端启动时，会向服务端发起请求，服务端接收到客户端的请求后，根据配置的仓库地址，将 git 上的文件克隆到本地的一个临时目录中，这个目录是一个 git 的本地仓库，然后服务端再读取本地文件，返回给客户端。这样做的好处是：当 git 服务故障或网络请求异常时，保证服务端依然能正常工作。

![Spring Cloud Config git 版工作原理](/images/spring-cloud/config/config-git.png)

---

# 入门案例

## config repo

使用 git 做配置中心的配置文件存储，需要一个 git 仓库，用于保存配置文件。 本例仓库地址： https://gitee.com/laiyy0728/config-repo

在仓库中，新建一个文件夹：`config-simple`，在文件夹内新建 3 个文件：`config-simple-dev.yml`、`config-simple-test.yml`、`config-simple-prod.yml`
![Spring Cloud Simple Config](/images/spring-cloud/config/config-repo-simple.png)

## config server

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-server</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

```yml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://gitee.com/laiyy0728/config-repo # git 仓库地址
          search-paths: config-simple # 从哪个文件夹下拉取配置
  application:
    name: spring-cloud-config-simple-server
server:
  port: 9090
```

```java
@SpringBootApplication
@EnableConfigServer
public class SpringCloudConfigSimpleServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringCloudConfigSimpleServerApplication.class, args);
    }
}
```

## 验证 config server

启动 config server，查看 endpoints mappings
![Spring Cloud Config Endpoints](/images/spring-cloud/config/config-endpoints.png)

- label：代表请求的是哪个分支，默认是 master 分支
- name：代表请求哪个名称的远程文件
- profile：代表哪个版本的文件，如：dev、test、prod 等

从 mappings 中，可以看出，访问获取一个配置的信息，有多种方式，尝试获取 `/config-simple/config-simple.dev.yml` 配置信息：
### 由接口获取配置详细信息
http://localhost:9090/config-simple/dev/master 、http://localhost:9090/config-simple/dev 
```json
{
	"name": "config-simple",
	"profiles": [
		"dev"
	],
	"label": "master",
	"version": "520b379e9c7f2e39bb56e599f914b6c08fe13c06",
	"state": null,
	"propertySources": [{
		"name": "https://gitee.com/laiyy0728/config-repo/config-simple/config-simple-dev.yml",
		"source": {
			"com.laiyy.gitee.config": "dev 环境，git 版 spring cloud config"
		}
	}]
}
```

### 由绝对文件路径获取配置文件内容

http://localhost:9090/master/config-simple-dev.yml 、http://localhost:9090/config-simple-dev.yml
```yml
com:
  laiyy:
    gitee:
      config: dev 环境，git 版 spring cloud config
```

## config client

在 config server 中获取配置文件以及成功，接下来需要在 config client 中，通过 config server 获取对应的配置文件

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-client</artifactId>
    </dependency>
</dependencies>
```


***application.yml***
```yml
spring:
  cloud:
    config:
      label: master
      uri: http://localhost:9090
      name: config-simple
      profile: dev
  application:
    name: spring-cloud-config-simple-client
server:
  port: 9091
```

```java
// 用于从远程 config server 获取配置文件内容
@Component
@ConfigurationProperties(prefix = "com.laiyy.gitee")
public class ConfigInfoProperties {
    private String config;

    public String getConfig() {
        return config;
    }

    public void setConfig(String config) {
        this.config = config;
    }
}


// 用于打印获取到的配置文件内容
@RestController
public class ConfigController {

    private final ConfigInfoProperties configInfoProperties;

    @Autowired
    public ConfigController(ConfigInfoProperties configInfoProperties) {
        this.configInfoProperties = configInfoProperties;
    }

    @GetMapping(value = "/get-config-info")
    public String getConfigInfo(){
        return configInfoProperties.getConfig();
    }

}


// 启动类
@SpringBootApplication
public class SpringCloudConfigSimpleClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudConfigSimpleClientApplication.class, args);
    }

}
```

## 验证 config client

启动 config client，观察控制台，发现 config client 拉取配置的路径是：http://localhost:8888 ，而不是在 yml 中配置的 localhost:9090。这是因为 boot 启动时加载配置文件的顺序导致的。boot 默认先加载 bootstrap.yml 配置，再加载 application.yml 配置。所以需要将 config server 配置移到 bootstrap.yml 中
![Config client default fetch server](/images/spring-cloud/config/config-client.png)

***bootstrap.yml***
```yml
spring:
  cloud:
    config:
      label: master  # 代表请求 git 哪个分支，默认 master
      uri: http://localhost:9090 # config server 地址
      name: config-simple # 获取哪个名称的远程文件，可以有多个，英文逗号隔开
      profile: dev # 代表哪个分支
```

***application.yml***
```yml
spring:
  application:
    name: spring-cloud-config-simple-client
server:
  port: 9091
```
![config client remote server](/images/spring-cloud/config/config-client-remote.png)


访问 http://localhost:9091/get-config-info
![get config info](/images/spring-cloud/config/get-config-info.png)