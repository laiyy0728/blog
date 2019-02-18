---
title: Spring Cloud 微服务（20） --- Zuul(一) <br> 基本概念、配置
date: 2019-02-13 16:12:03
updated: 2019-02-13 16:12:03
categories:
    Java
tags:
    - SpringCloud
    - Zuul
---


Zuul 是由 Netflix 孵化的一个致力于“网关”的解决方案的开源组件。Zuul 在动态路由、监控、弹性、服务治理、安全等方面有着举足轻重的作用。
Zuul 底层为 Servlet，本质组件是一系列 Filter 构成的责任链。

<!-- more -->

**Zuul 具备的功能**

- 认证、鉴权
- 压力控制
- 金丝雀测试（灰度发布）
- 动态路由
- 负载削减
- 静态响应处理
- 主动流量控制

# Zuul 入门案例

## Zuul Server

***Server源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-zuul/spring-cloud-zuul-simple***
***Client源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-zuul/spring-cloud-zuul-provider-service-simple***

### pom、yml
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
</dependencies>
```

```yml
spring:
  application:
    name: spring-cloud-zuul-simple
server:
  port: 8989
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    instance-id: ${spring.application.name}:${server.port}
    prefer-ip-address: true
zuul:
  routes:  # zuul 路由配置，map 结构
    spring-cloud-provider-service-simple: # 针对哪个服务进行路由
      path: /provider/**  # 路由匹配什么规则。 当前配置为 provider 开头的请求路由到 provider-service 上
      # serviceId: spring-cloud-provider-service-simple # 路由到哪个 serviceId 上（即哪个服务），可不设置
```

### 启动类
```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableZuulProxy        // 开启 Zuul 代理
public class SpringCloudZuulSimpleApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudZuulSimpleApplication.class, args);
    }

}
```

## 验证

先请求 provider-service： http://localhost:8081/get-result

![Zuul Provider Service](/images/spring-cloud/zuul/provider-service.png)

再请求 zuul server： http://localhost:8989/provider/get-result
![Zuul Server Provider](/images/spring-cloud/zuul/zuul-server-provider-service.png)

可以看到，响应结果一致，但通过 `Zuul Server` 的请求路径多了 `/provider`，由此验证 zuul server 路由代理成功

---

# 典型配置

在上例中，路由规则的配置
```yml
zuul:
  routes:  
    spring-cloud-provider-service-simple: 
      path: /provider/**  
      serviceId: spring-cloud-provider-service-simple 
```

实际上，可以将这个配置进行简化

## 指定路由的简化

```yml
zuul:
  routes:  
    spring-cloud-provider-service-simple: /provider/**  
```

## 默认简化

默认简化可以不指定路由规则：

```yml
zuul:
  routes:  
    spring-cloud-provider-service-simple: 
```

此时简化配置，相当于：
```yml
zuul:
  routes:  
    spring-cloud-provider-service-simple: 
      path: /spring-cloud-provider-service-simple/**
      serviceId: spring-cloud-provider-service-simple
```

## 多实例路由

一般情况下，一个服务会有多个实例，此时需要对这个服务进行负载均衡。默认情况下，Zuul 会使用 Eureka 中集成的基本负载均衡功能（轮询）。

如果需要使用 Ribbon 的负载均衡功能，有两种方式：

### Ribbon 脱离 Eureka 使用

需要在 `routes` 配置中指定 `serviceId`，这个操作需要禁止 Ribbon 使用 Eureka。
***此方式必须指定 serviceId***
```yml
zuul:
  routes:
    spring-cloud-provider-service-simple: # 服务名称，需要和下方配置一致
      path: /ribbon-route/**
      serviceId: spring-cloud-provider-service-simple   # serviceId，需要和下方配置一致
ribbon:
  eureka:
    enabled: false    # 禁用掉 Eureka

spring-cloud-provider-service-simple:     # 服务名称，需要和上方配置一致
  ribbon:
    NIWSServerListClassName: com.netflix.loadbalancer.ConfigurationBasedServerList  # 设置 ServerList 的配置
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule    # 设置负载均衡策略
    listOfServers: http://localhost:8080,http://localhost:8081    # 负载的 server 列表
```

### Ribbon 不脱离 Eureka 使用

直接使用 ribbon 路由配置即可
***此方式可以不指定 serviceId***
```yml
zuul:
  routes:
    spring-cloud-provider-service-simple:
      path: /ribbon-route/**

spring-cloud-provider-service-simple:
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule
```

## Zuul 本地跳转

如果在 zuul 中做一些逻辑处理，在访问某个接口时，跳转到 zuul 中的这个方法上来处理，就需要用到 zuul 本地跳转

```yml
zuul:
  routes:
    spring-cloud-provider-service-simple:
      path: /provider/**  # 只有访问 /provider 的时候才会 forward，但凡后面多一个路径就不行了。。。 为啥。。。
      url: forward:/client
```

此时，访问：http://localhost:8989/client ，可以访问到，访问 http://localhost:8989/provider ，也能访问到，如果访问 http://localhost:8989/provider/get-result ，理论上应该也能跳转到 /client，但是实际上会报 404 错误
```json
{
    "timestamp": "2019-02-15T06:50:12.565+0000",
    "status": 404,
    "error": "Not Found",
    "message": "No message available",
    "path": "/provider/get-result"
}
```

如果去掉 `url: forward:/client`，再访问 http://localhost:8989/provider/get-result ，结果正常：
```json
```

## Zuul 相同路径加载规则

```yml
zuul:
  routes:
    spring-cloud-provider-service-simple-a:
      path: /provider/**
      serviceId: spring-cloud-provider-service-simple-a
    spring-cloud-provider-service-simple-b:
      path: /provider/**
      serviceId: spring-cloud-provider-service-simple-b
```

可以发现，/provider/** 匹配了2个 serviceId，这个匹配结果只会路由到最后一个服务上。即：/provider/** 只会被路由到 simple-b 服务上。
yml 解释器在工作时，如果同一个映射路径对应了多个服务，按照加载顺序，后面的规则会把前面的规则覆盖掉。


## 路由通配符

| 规则 | 解释 | 示例 |
| :-: | :-: | :-: |
| /** | 匹配任意数据量的路径与字符 | /client/aa，/client/aa/bb/cc | 
| /* | 匹配任意数量的字符 | /client/aa，/client/aaaaaaaaaaaaaa |
| /? | 匹配单个字符 | /client/a，/client/b，/client/c |


---

# 功能配置

## 路由配置

在配置路由规则时，可以配置一个统一的前缀
```yml
zuul:
  routes:
    spring-cloud-provider-service-simple:
      path: /provider/**
      serviceId: spring-cloud-provider-service-simple
  prefix: /api
```

访问 http://localhost:8989/provider/get-result 
```json
{
    "timestamp": "2019-02-15T07:16:46.625+0000",
    "status": 404,
    "error": "Not Found",
    "message": "No message available",
    "path": "/provider/get-result"
}
```

访问 http://localhost:8989/api/provider/get-result 
```
this is provider service! this port is: 8081
```

这样的设置，会将每个访问访问前都加上 prefix 前缀，但是实际上访问的是 `path` 配置的路径。
如果某个服务不需要前缀，访问路径就是 prefix + path，则只需要在对应的服务配置设置 `stripPrefix: false` 即可

```yml
zuul:
  routes:
    spring-cloud-provider-service-simple:
      path: /provider/**
      serviceId: spring-cloud-provider-service-simple
      stripPrefix: false
  prefix: /api
```

此时访问： http://localhost:8989/pre/provider/get-result ，返回值为：
```json
{
    "timestamp": "2019-02-15T07:24:33.271+0000",
    "status": 404,
    "error": "Not Found",
    "message": "No message available",
    "path": "/pre/provider/get-result"
}
```

对比两个 404 错误，可以看到，同样是访问 /pre/provider/get-result，没有设置 `stripPrefix: false` 时，path 为 `/provider/get-result`，设置 `stripPrefix: false` 时，path 为 `/pre/provider/get-result`。即：设置 `stripPrefix: false` 时，请求路径和实际路径是一致的。


## 服务屏蔽、路径屏蔽

为了避免某些服务、路径被侵入，可以将其屏蔽掉
```yml
zuul:
  routes:
    spring-cloud-provider-service-simple:
      path: /provider/**
      serviceId: spring-cloud-provider-service-simple
  ignored-services: spring-cloud-provider-service-simple    # 此配置会在 zuul 路由时，忽略掉该服务
  ignored-patterns: /**/get-result/**   # 此配置会在 zuul 路由时，忽略掉可以匹配的路径
  prefix: /pre
```

## 敏感头信息

正常访问时，provider-service 接收到的 headers 为：
```
cache-control: no-cache
user-agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/71.0.3578.98 Safari/537.36
postman-token: 43dbfb6e-f529-543e-99d9-5d0a06bf79e6
accept: */*
accept-encoding: gzip, deflate, br
accept-language: zh-CN,zh;q=0.9
x-forwarded-host: localhost:8989
x-forwarded-proto: http
x-forwarded-prefix: /provider
x-forwarded-port: 8989
x-forwarded-for: 0:0:0:0:0:0:0:1
content-length: 0
host: 10.10.10.141:8081
connection: Keep-Alive
```

设置敏感头：
```yml
zuul:
  routes:
    spring-cloud-provider-service-simple:
      path: /provider/**
      serviceId: spring-cloud-provider-service-simple
      sensitiveHeaders: postman-token,x-forwarded-for,Cookie
```

此时再次访问，获取 headers
```
cache-control: no-cache
user-agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/71.0.3578.98 Safari/537.36
accept: */*
accept-encoding: gzip, deflate, br
accept-language: zh-CN,zh;q=0.9
x-forwarded-host: localhost:8989
x-forwarded-proto: http
x-forwarded-prefix: /provider
x-forwarded-port: 8989
content-length: 0
host: 10.10.10.141:8081
connection: Keep-Alive
```

对比发现，`sensitiveHeaders` 配置的 headers 在 provider-service 中已经接收不到了。
默认情况下，`sensitiveHeaders` 会忽略三个 header：`Cookie`、`Set-Cookie`、`Authorization`