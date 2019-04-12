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