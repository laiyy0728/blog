---
title: Spring Cloud 微服务（18） --- Actuator(一)
date: 2019-02-12 16:53:00
updated: 2019-02-12 16:53:00
categories:
    Java
tags:
    - SpringBoot
    - Actuator
---

在 Hystrix Dashboard 中，使用了 /actuator/hystrix.stream，查看正在运行的项目的运行状态。 其中 `/actuator` 代表 SpringBoot 中的 Actuator 模块。该模版提供了很多生产级别的特性，如：监控、度量应用。Actuator 的特性可以通过众多的 REST 端点，远程 shell、JMX 获得。

<!-- more -->

# 常见 Actuator 端点

Boot 中提供了 13 个基础的 Actuator 端点

*路径省略前缀：/actuator*

| 路径 | HTTP 动作 | 描述 |
| :-: | :-: | :-: |
| /autoconfig | GET | 提供了一份自动配置报告，记录哪些自动配置条件通过了，哪些没通过 |
| /configprops | GET | 描述配置属性（包含默认值）如何注入 Bean |
| /beans | GET | 描述应用程序上下文全部 bean，以及他们的关系 |
| /dump | GET | 获取线程活动快照 |
| /env | GET | 获取全部环境属性 |
| /env/{name} | GET | 获取指定名称的特定环境的属性值 |
| /health | GET | 报告应用长须的健康指标，由 HealthIndicator 的实现提供 |
| /info | GET | 获取应用程序定制信息，由 Info 开头的属性提供 |
| /mappings | GET | 描述全部的 URL 路径，以及它们和控制器(包含Actuator端点)的映射关系 |
| /metrics | GET | 报告各种应用程序度量信息，如：内存用量、HTTP 请求计数 |
| /metrics/{name} | GET | 根据名称获取度量信息 |
| /trace | GET | 提供基本的 HTTP 请求跟踪信息(时间戳、HTTP 头等)|
| /shutdown | POST | 关闭应用，要求 endpoints.shutdown.enabled 为 true |


SpringCloud 中，默认开启了 `/actuator/health`、`/actuator/info` 端点，其他端点都屏蔽掉了。如果需要开启，自定义开启 endpoints 即可
```yml
management:
  endpoints:
    web:
      exposure:
        exclude: ['health','info','beans','env']
```
如果要开启全部端点，设置 `exclude: *` 即可