---
title: Spring Cloud 微服务（35） --- SpringCloud Gateway（一） 基础概念
date: 2019-04-18 16:40:59
updated: 2019-04-18 16:40:59
categories: spring-cloud
tags: [SpringCloud, Gateway]
---

SpringCloud Gateway 是 Spring 官方基于 Spring 5.0、SpringBoot 2.0、Project Reactor 等技术开发的网关，SpringCloud Gateway 旨在为微服务架构提供简单、有效、且统一的 API 路由管理方式。SpringCloud Gateway 作为 Spring Cloud 生态系统中的网关，目标是替代 Netflix Zuul，其不仅提供统一的路由方式，并且还基于 Filter 链的方式提供了网关的基本功能：安全、监控、埋点、限流等

<!-- more -->
---

# SpringCloud Gateway 核心概念

网关提供 API 全托管服务，丰富的 API 管理功能，辅助企业管理大规模 API，以降低管理成本和安全风险，包括协议适配、协议转发、安全策略、防刷、流量、监控日志等功能。一般来说，网关对外暴露的 URL 或者接口信息，我们统称为路由信息。网关的核心是 Filter 以及 FIlter Chain。

SpringCloud Gateway 中的几个概念：
- 路由：网关最基本的部分，路由信息由一个 ID、一个目的 URL、一组断言工厂、一组 Filter 组成。如果路由断言为真，则说明请求的 url 和配置的路由匹配。
- 断言：Java8 中的断言函数。SpringCloud Gateway 中的断言函数输入类型的 Spring 5.0 框架中的 ServerWebExchange。SpringCloud Gateway 中的断言函数允许开发者去匹配来自于 Http Request 中的任何信息，比如请求头、参数等
- 过滤器：一个标准的 Spring WebFilter，SpringCloud Gateway 中的 Filter 分为两种类型：GatewayFilter、GlobalFilter。Filter 将对请求和响应进行修改处理。


---

# 工作原理

***示意图***
![gateway](/images/spring-cloud/gateway/gateway.png)


Gateway 的客户端会向 SpringCloud Gateway 发起请求，请求会首先被 HTTPWebHandlerAdapter 进行提取组装成网关的上下文，然后网关的上下文会传递到 DispatcherHandler。DispatcherHandler 是所有请求的分发处理器，DispatcherHandler 主要负责分发请求对应的处理器，如：将请求分发到 RoutePredicateHandlerMapping（路由断言处理映射器）。路由断言处理映射器主要用于路由的查找，以及找到路由后返回对应的 FilterWebHandler。FilterWebHandler 主要负责组装 Filter 链表，并调用 Filter 执行一系列的 Filter 处理，然后把请求转到后端对应的代理服务处理，处理完毕后，将 Response 返回到 Gateway 客户端。

在 Filter 链中，通过虚线分割 Filter 的原因是，过滤器可以在转发请求之前处理或接收到被代理服务的返回结果之后处理。所有的 pre 类型的 Filter 执行完毕之后，才会转发请求到被代理的服务处理。被代理的服务吧所有请求处理完毕后，才会执行 Post 类型的过滤器。

***注意：在配置路由时，如果不指定端口，http 默认为 80，https 默认是 443；Gateway 启动容器暂时只支持 Netty。***


---

# 实例

目标：使用 Gateway 的 Path 断言工厂，实现 url 的转发

由于 SpringCloud Gateway 的容器是 Netty，而不是 tomcat，所有 pom 中不能存在 `spring-boot-starter-web` 依赖。

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>
</dependencies>
```

```yml
server:
  port: 8080

spring:
  application:
    name: spring-cloud-gateway-simple-1

logging:
  level:
    org.springframework.cloud.gateway: TRACE
    org.springframework.http.server.reactive: debug
    org.springframework.web.reactive: debug
    reactor.ipc.nettu: debug
management:
  endpoints:
    web:
      exposure:
        include: '*'
```

使用 API 形式创建路由断言
```java
@SpringBootApplication
public class SpringCloudGatewaySimple1Application {


    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder routeLocatorBuilder) {
        return routeLocatorBuilder.routes().route(r -> r.path("/jd")
                .uri("https://jd.com:80")
                .id("jd_route")
        ).build();
    }

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudGatewaySimple1Application.class, args);
    }

}
```

使用 yml 形式创建路由断言
```yml
spring:
  cloud:
    gateway:
      routes:
        - id: baidu_route
          uri: http://baidu.com
          predicates:
            - Path=/baidu
```

启动验证：访问 http://localhost:8080/jd 、http://localhost:8080/baidu 自动跳转到 京东、百度首页。验证成功。

## 启动时可能出现的问题：

```

***************************
APPLICATION FAILED TO START
***************************

Description:

Parameter 0 of method modifyRequestBodyGatewayFilterFactory in org.springframework.cloud.gateway.config.GatewayAutoConfiguration required a bean of type 'org.springframework.http.codec.ServerCodecConfigurer' that could not be found.


Action:

Consider defining a bean of type 'org.springframework.http.codec.ServerCodecConfigurer' in your configuration.

Disconnected from the target VM, address: '127.0.0.1:6111', transport: 'socket'

Process finished with exit code 1
```

错误原因：项目中存在 tomcat 依赖，需要去掉 `spring-boot-starter-web` 依赖  

---

# 断言器

## After

After 路由断言器中，会取一个 UTC 时间格式的时间参数，当请求进来的当前时间，在配置的 UTC 时间之后，则会匹配成功，否则不不能匹配成功。

API 形式
```java
@Bean
    public RouteLocator afterRouteLocator(RouteLocatorBuilder routeLocatorBuilder){
        ZonedDateTime zonedDateTime = LocalDateTime.now().minusSeconds(20).atZone(ZoneId.systemDefault());
        return routeLocatorBuilder.routes()
                .route("after_route", route -> route.after(zonedDateTime).uri("http://baidu.com"))
                .build();
    }
```

yml 形式
```yml
spring:
  cloud:
    gateway:
      routes:
        - id: after_route
          uri: http://baidu.com
          predicates:
            - After=2019-04-17T22:04:04.888+08:00[Asia/Shanghai]
```

## Before

Before 断言工厂会取一个 UTC 格式的时间参数，当请求进来的当前时间，在路由断言其工厂之前，会成功匹配，否则不会匹配成功。

API 形式
```java
@Bean
    public RouteLocator afterRouteLocator(RouteLocatorBuilder routeLocatorBuilder){
        ZonedDateTime zonedDateTime = LocalDateTime.now().plusMonths(1).atZone(ZoneId.systemDefault());
        return routeLocatorBuilder.routes()
                .route("before_route", route -> route.before(zonedDateTime).uri("http://baidu.com"))
                .build();
    }
```

yml 形式
```yml
spring:
  cloud:
    gateway:
      routes:
        - id: after_route
          uri: http://baidu.com
          predicates:
            - Before=2019-04-18T22:13:12.556+08:00[GMT+08:00]
```

## Between

Between 断言工厂会取一个 UTC 时间格式的时间参数，当请求进来的当前时间在配置的 UTC 时间工厂之间会匹配成功，否则不能成功匹配。

API 形式
