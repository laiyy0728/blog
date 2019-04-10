---
title: Spring Cloud 微服务（31） --- Spring Cloud Config(五) <BR> 客户端安全认证 JWT
date: 2019-03-07 10:29:50
updated: 2019-03-07 10:29:50
categories:
    Java
tags:
    [SpringCloud, CloudConfig]
---

Spring Cloud Config 客户端使用 JWT 身份验证方法代替标准的基本身份验证，这种方式需要对服务端、客户端进行改造
- 客户端向服务端授权 Rest Controller 发送请求并带上用户名、密码
- 服务端返回 JWT Token
- 客户端查询服务端的配置需要在 header 中带上 Token 令牌进行认证


<!-- more -->

***源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-config/spring-cloud-config-jwt***