---
title: Spring Cloud 微服务（20） --- Zuul <br>
date: 2019-02-13 16:12:03
updated: 2019-02-13 16:12:03
categories:
    Java
tags:
    - SpringCloud
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