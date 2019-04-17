---
title: Spring Cloud 微服务（32） --- APM(二) <BR> SkyWalking
date: 2019-04-12 16:59:07
updated: 2019-04-12 16:59:07
categories: Java
tags: [SpringCloud, APM]
---

SkyWalking 是有个完整的 APM 系统，被用于追踪、监控、诊断分布式系统。

SkyWalking 整体由 4 个部分组成：`collector`、`agent`、`web`、`storage`。
应用级别的接入，可以使用 SDK 形式接入，也可以使用非侵入式的 `Agent` 形式接入。`agent` 将数据转化为 SkyWalking Trace 数据协议，通过 HTTP、gRPC 发送到 `collector`，`collector` 对收集到的数据进行分析、整合，最后存储到 es 或 H2 中，一般情况下，H2 用于测试。

<!-- more -->

# SkyWalking 特性

## SkyWalking 主要功能

- 分布式只追踪、上下文传输
- 应用、实例、服务性能指标分析
- 根源分析
- 应用拓扑分析
- 应用于服务依赖分析
- 慢服务检测
- 性能优化

## SkyWalking 主要特性

- 多语言探针、类库
 1. Java 自动探针，追踪、监控程序时，无需修改源码
 2. 社区提供多语言探针：.NET、Node.js
- 多种后端存储：Elasticsearch、H2 等
 1. 支持 OpenTrancing：Java 自动探针和 OpenTracing API 协同工作
- 轻量级、完善的后台聚合和分析功能
- 现代化 Web UI
- 日志集成
- 应用、实例、服务的告警
- 支持接受其他跟踪器数据格式
 1. Zipkin JSON、Thrift、Protobuf v1 和 v2 格式，由 OpenZipkin 库提供支持
 2. Jaeger 采用 Zipkin Thrift 或 JSON v1/v2 格式


 --- 

 # SkyWalking 测试用例代码

 ## Zuul

 ```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
    </dependency>
</dependencies>
 ```

 ```yml
 spring:
  application:
    name:
      spring-cloud-apm-skywalking-zuul
server:
  port: 9020
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
zuul:
  routes:
    spring-cloud-apm-skywlaking-consumer:
      path: /client/**
      serviceId: spring-cloud-apm-skywlaking-consumer

ribbon:
  eureka:
    enabled: true
  ReadTimeout: 30000
  ConnectionTimeout: 30000
  MaxAutoRetries: 0
  MaxAutoRetriesNextServer: 1
  OkToRetryOnAllOperations: false

hystrix:
  threadpool:
    default:
      coreSize: 1000
      maxQueueSize: 1000
      queueSizeRejectionThreshold: 500

  command:
    default:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 120001
 ```

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableZuulProxy
public class SpringCloudApmSkywalkingZuulApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudApmSkywalkingZuulApplication.class, args);
    }

}
```

## Consumer

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
</dependencies>
```

```yml
server:
  port: 9021
spring:
  application:
    name: spring-cloud-apm-skywlaking-consumer
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

```java
@SpringBootApplication
@EnableFeignClients
@EnableDiscoveryClient
public class SpringCloudApmSkywlakingConsumerApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudApmSkywlakingConsumerApplication.class, args);
    }

}



@FeignClient("spring-cloud-apm-skywalking-provider")
public interface SkyWalkingFeignService {

    @RequestMapping(value = "/get-send-info", method = RequestMethod.GET)
    String getSendInfo(@RequestParam("serviceName") String serviceName);

}


@RestController
public class SkyWalkingController {

    private final SkyWalkingFeignService feignService;

    @Autowired
    public SkyWalkingController(SkyWalkingFeignService feignService) {
        this.feignService = feignService;
    }

    @GetMapping(value = "/get-info")
    public String getInfo(){
        return feignService.getSendInfo("service");
    }

}
```

## Provider

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
</dependencies>
```


```yml
server:
  port: 9022
spring:
  application:
    name: spring-cloud-apm-skywalking-provider
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

```java
@SpringBootApplication
@EnableDiscoveryClient
@RestController
public class SpringCloudApmSkywalkingProviderApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudApmSkywalkingProviderApplication.class, args);
    }

    @GetMapping(value = "/get-send-info")
    public String getSendInfo(@RequestParam("serviceName") String serviceName){
        return serviceName + " --> " + "spring-cloud-apm-skywalking-provider";
    }

}
```

---

# SkyWalking 安装

SkyWalking 依赖环境：
- 被监控的应用运行在 JDK6+
- SkyWalking collector 和 WebUI 运行在 JDK8+
- elasticsearch 5.x（集群可能不能使用）

## 下载 elasticsearch_5.6.10 版本

***注意：一定要用 5.x 的 elasticsearch，否则会出现版本问题！***

解压安装后，进入 `config/elasticsearch.yml` 文件，修改 `network.host` 为 `0.0.0.0`。elasticsearch 不允许 root 用户启动，建立新用户并赋权：
```shell
useradd es
chown -R es:es /path/to/es
```

切换到 es 用户，启用 es
```
./bin/elasticsearch
```

控制台报错：
```
[1]: max file descriptors [4096] for elasticsearch process is too low, increase to at least [65536]
[2]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
```

修改 `/etc/security/limits.conf`，增加如下配置：
```
* soft nofile 819200
* hard nofile 819200
```

修改 `/etc/sysctl.conf`，增加配置：
```
vm.max_map_count=655360
```

重新加载配置：
```
sysctl -p
```

后台启动 es： `./bin/elasticsearch -d`

查看 es 日志：`tail -f logs/elasticsearch.log`，正常启动。

浏览器访问 elasticsearch，可见，默认的 cluster name 为 `elasticsearch`：
![elasticsearch](/images/spring-cloud/apm/elasticsearch.png)

## SkyWalking 目录结构

![SkyWalking 目录结构](/images/spring-cloud/apm/skywalking-dir-tree.png)

- agent：探针相关
- bin：collectorService、webappService 启动脚本，其中 startup.* 是同事启动两个脚本的合并命令
- config：Collector 的相关配置信息
- log：collector、web 的日志文件
- webapp：存放 SkyWalking 展示 UI 的 jar 和配置文件

SkyWalking 的默认端口为：8080、10800、11800、12800 等，如果要修改端口，需要修改 config 目录下的 application.yml、webapp 下的 webapp.yml。

修改 `config/application.yml` 文件，clusterName 与 elasticsearch 的 cluster name 一致，其余采用默认设置。
![application.yml](/images/spring-cloud/apm/application.yml.png)

## 启动 SkyWalking

启动 SkyWalking：
```shell
./bin/startup.sh
```

![启动 SkyWalking](/images/spring-cloud/apm/startup-success.png)

访问 SkyWalking：
![访问 SkyWalking](/images/spring-cloud/apm/skywalking-login-page.png)

默认 用户名/密码 为：admin/admin

![SkyWalking Dashboard](/images/spring-cloud/apm/skywalking-dashboard.png)

---

# 监控项目

## 创建目录

创建四个目录，分别对应：consumer、provider、zuul、eureka-server 四个应用，每个应用使用一个对应的 agent 进行启动，其中 agent 是 SkyWalking 的 agent 目录。

修改 `agent.config` 文件中 `agent.application_code`，这项配置代表应用。对应修改为 `consumer`、`provider`、`zuul`、`eureka`。将 eureka、zuul、consumer、provider 打包为 jar，上传到对应目录中。

## 修改 es 内存配置

elasticsearch 默认 JVM 内存为 2g，如果虚拟机内存过小，无法启动。如果略大于 JVM 内存，启动后无法启动其他组件。所以需要修稿 elasticsearch 的默认 JVM 内存。修改 `$ES_HOME/config/jvm.options`:
![es jvm 参数](/images/spring-cloud/apm/es-jvm.png)

修改后重启 es、SkyWalking

使用 `top` 命令查看使用内存最高的应用，使用 `free` 命令，查看内存总用量、剩余内存。
![free top 命令](/images/spring-cloud/apm/free-top.png)

## 依次启动四个应用

启动时需要指定 JVM 内存，防止出现内存不够的情况。 
`-Xms` 指定最小内存，`-Xmx` 指定最大内存

eureka
`java -javaagent:/usr/local/src/soft/eureka/agent/skywalking-agent.jar -jar /usr/local/src/soft/eureka/spring-cloud-eureka-server-simple-0.0.1-SNAPSHOT.jar -Xms256m -Xmx256m`

provider
`java -javaagent:/usr/local/src/soft/provider/agent/skywalking-agent.jar -jar /usr/local/src/soft/provider/spring-cloud-apm-skywalking-provider-0.0.1-SNAPSHOT.jar -Xms256m -Xmx256m`

consumer
`java -javaagent:/usr/local/src/soft/consumer/agent/skywalking-agent.jar -jar /usr/local/src/soft/consumer/spring-cloud-apm-skywalking-consumer-0.0.1-SNAPSHOT.jar -Xms256m -Xmx256m`

zuul
`java -javaagent:/usr/local/src/soft/zuul/agent/skywalking-agent.jar -jar /usr/local/src/soft/zuul/spring-cloud-apm-skywalking-zuul-0.0.1-SNAPSHOT.jar -Xms256m -Xmx256m`

## 确认启动成功

使用 `jps` 命令查看启动进程：
![jps](/images/spring-cloud/apm/jps.png)

查看剩余内存是否满足正常运行：
![free](/images/spring-cloud/apm/free.png)

## 验证 SkyWalking

启动成功后访问eureka： http://192.168.67.135:8761/
![eureka](/images/spring-cloud/apm/eureka.png)

访问 SkyWalking： 
![SkyWalking](/images/spring-cloud/apm/skywalking-dashboard-1.png)

可见 4 个 app 都启动成功了。使用 zuul 访问 consumer，调用 provider： 
![zuul](/images/spring-cloud/apm/zuul.png)

再次查看 SkyWalking：
![SkyWalking](/images/spring-cloud/apm/skywalking-dashboard-2.png)

在 service 选项卡中可以看到每个 service 的具体调用情况
![service](/images/spring-cloud/apm/service.png)