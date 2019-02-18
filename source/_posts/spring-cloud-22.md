---
title: Spring Cloud 微服务（22） --- Zuul(三) <br> Zuul 权限集成
date: 2019-02-18 14:27:07
updated: 2019-02-18 14:27:07
categories:
    Java
tags:
    - SpringCloud
    - Zuul
---

在原本的单体应用中，通常使用 Apache Shiro、Spring Security 等权限框架，但是在 Spring Cloud 中，面对成千上万的微服务，而且每个服务之间无状态，使用 Shiro、Security 难免力不从心。在解决方案的选择上，传统的单点登录SSO、分布式 session 等，要么致使权限服务器集中化，导致流量臃肿，要么需要实现一套复杂的存储同步机制，都不是最好的解决方案。

<!-- more -->

可以使用 Spring Cloud Zuul 自定义实现权限认证方式

***源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-zuul/spring-cloud-zuul-security***

---



# 自定义权限认证

## Filter

Zuul 对于请求的转发是通过 Filter 链控制的，可以在 RequestContext 的基础上做任何事。所以只需要在 `spring-cloud-zuul-filter` 的基础上，设置一个执行顺序比较靠前的 Filter，就可以专门用于对请求特定内容做权限认证。

优点：实现灵活度高，可整合已有的权限系统，对原始系统违法化友好
缺点：需要开发一套新的逻辑，维护成本增加，调用链紊乱


## OAuth2.0 + JWT

OAuth2.0 是对于“授权-认证”比较成熟的面向资源的授权协议。整个授权流程中，用户是资源拥有者，服务端需要资源拥有者的授权，这个过程相当于键入密码或者其他第三方登录。触发了这个操作后，客户端就可以向授权服务器申请 Token，拿到后，再携带 Token 到资源所在服务器拉取响应资源。

JWT(JSON Web Token)是一种使用 JSON 格式来规范 Token 或 Session 的协议。由于传统认证方式会生成一个凭证，这个凭证可以是 Token 或 Session，保存于服务端或其他持久化工具中，这样一来，凭证的存取或十分麻烦。JWT 实现了“客户端 Session”。

JWT 的组成部分：
- Header 头部：指定 JWT 使用的签名算法
- Payload 载荷：包含一些自定义与非自定义的认证信息
- Signature：将头部、载荷使用“.”连接后，使用头部的签名算法生成签名信息，并拼装到末尾

OAuth2.0 + JWT 的意义在于，使用 OAuth2.0 协议思想拉取认证生成 TToken，使用 JWT 瞬时保存这个 Token，在客户端与资源端进行对称或非对称加密，是的这个规约具有定时、定量的授权认证功能，从而免去 Token 存储带来的安全或者系统扩展问题。


## 实现

### Zuul Server

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
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-oauth2</artifactId>
        <exclusions>
            <exclusion>
                <!-- spring-security-oauth2-autoconfigure 在 spring-cloud-starter-oauth2 中默认是 2.1.0.M4 版本，这个版本可能会出现 jar 包下载失败的问题 -->
                <!-- 因此将此版本剔除，使用 2.1.0.RELEASE 版本 -->
                <groupId>org.springframework.security.oauth.boot</groupId>
                <artifactId>spring-security-oauth2-autoconfigure</artifactId>
            </exclusion>
        </exclusions>
    </dependency>
    <!-- 使用 2.1.0.RELEASE 版本的 spring-security-oauth2-autoconfigure -->
    <dependency>
        <groupId>org.springframework.security.oauth.boot</groupId>
        <artifactId>spring-security-oauth2-autoconfigure</artifactId>
        <version>2.1.0.RELEASE</version>
    </dependency>
</dependencies>
```

```yml
spring:
  application:
    name: spring-cloud-zuul-security-server
server:
  port: 5555
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
    instance-id: ${spring.application.name}:${server.port}
zuul:
  routes:
    spring-cloud-zuul-security-provider-service:
      path: /provider/**
      serviceId: spring-cloud-zuul-security-provider-service
security:
  oauth2:
    client:
      access-token-uri:  http://localhost:7777/uaa/oauth/token # 令牌端点
      user-authorization-uri: http://localhost:7777/uaa/oauth/authorize # 授权端点
      client-id: zuul_server # OAuth2 客户端id
      client-secret: secret # OAuth2 客户端秘钥
    resource:
      jwt:
        key-value: spring-cloud # 使用对称加密，默认算法为 HS256，加密秘钥为 spring-cloud
```

### auth server

```xml
 <dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-oauth2</artifactId>
        <exclusions>
            <exclusion>
                <groupId>org.springframework.security.oauth.boot</groupId>
                <artifactId>spring-security-oauth2-autoconfigure</artifactId>
            </exclusion>
        </exclusions>
    </dependency>
    <dependency>
        <groupId>org.springframework.security.oauth.boot</groupId>
        <artifactId>spring-security-oauth2-autoconfigure</artifactId>
        <version>2.1.0.RELEASE</version>
    </dependency>
</dependencies>
```