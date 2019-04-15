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