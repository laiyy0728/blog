---
title: Spring Cloud 微服务（34） --- APM(三) <BR> Pinpoint
date: 2019-04-17 16:38:55
updated: 2019-04-17 16:38:55
categories: spring-cloud
tags: [SpringCloud, APM]
---

`Pinpoint` 是韩国人编写的 APM 系统，是一个分析大规模分布式系统的平台，并提供处理大量跟踪数据的解决方案。


<!-- more -->

# Pinpoint

## 特点

- 分布式事务追踪，跟踪跨分布式应用的消息
- 自动检测应用拓展
- 水平扩展，以便支持大规模服务器集群
- 提供代码级了践行，便于定位失败点和瓶颈
- 提供字节码增强技术，添加新功能无需修改代码

## 优势

- 非侵入式：使用字节码增强技术，添加新功能无需修改代码
- 资源消耗小：对性能影响最小（资源使用量增加约3%）

## 架构模块

- HBase：主要用于存储数据
- Pinpoint Collector：部署在 Web 容器上
- Pinpoint Web：部署在 Web 容器上
- Pinpoint Agent：附加到用于分析的 Java 应用程序

流程：首先通过 agent 收集调用应用的数据，将数据发送到 collector，collector 通过处理和分析数据，最后存储到 HBase 中，可以通过 Pinpoint Web UI 查看已经分析好的调用分析数据

## 数据结构

- Span：RPC 跟踪的基本单位，表示 RPC 到达时处理的工作，包含跟踪数据。Span 将子项标记未 SpanEvent，作为数据结构，每个 Span 包含一个 TraceId
- Trace：一系列跨度，由相关的 RPC(Span) 组成。同一跟踪中的跨距共享相同的 TransactionId。Trace 通过 SpanIds 和 ParentSpanIds 排序为分层树结构
- TraceId：由 TransactionId、SpanId、ParentSpanId 组成的秘钥集合。`TransactionId` 代表消息id，SpanId 和 ParentSpanId 表示 RPC 父子关系
> TransactionId：来自单个事务的分布式系统发送、接收的消息id，必须在整个服务器组是全局唯一的
> SpanId：接收 RPC 消息时处理的作业 ID，是在 RPC 到达节点时生成的
> ParentSpanId：生成 RPC 的父 span 的 spanId，如果节点是事务的起始点，不会有父跨度。

![pinpont 架构图](/images/spring-cloud/apm/pinpoint.png)

## 兼容性

### JDK 兼容性
| Pinpoint 版本 | Agent 需要的 JDK 版本 | Collector 需要的 JDK 版本 | Web 需要的 JDK 版本 |
| :-: | :-: | :-: | :-: |
| 1.0.x | 6-8 | 6+ | 6+ |
| 1.1.x | 6-8 | 7+ | 7+ |
| 1.5.x | 6-8 | 7+ | 7+ |
| 1.6.x | 6-8 | 7+ | 7+ |
| 1.7.x | 6-8 | 8+ | 8+ |
| 1.8.x | 6-8,9+ | 8+ | 8+ |


### Base 兼容性

| Pinpoint 版本 | HBase 0.94.x | HBase 0.98.x | HBase 1.0.x | HBase 1.1.x | HBase 1.2.x |
| :-: | :-: |  :-: |  :-: |  :-: |  :-: | 
| 1.0.x | √ | × | × | × | × |
| 1.1.x | × | not tested | √ | not tested | not tested |
| 1.5.x | × | not tested | √ | not tested | not tested |
| 1.6.x | × | not tested | not tested | not tested | √ |
| 1.7.x | × | not tested | not tested | not tested | √ |
| 1.8.x | × | not tested | not tested | not tested | √ |

### Agent-Collector 兼容性

| Agent 版本 | Collector 1.0.x | Collector 1.1.x | Collector 1.5.x | Collector 1.6.x | Collector 1.7.x | Collector 1.8.x |
| :-: |:-: |:-: |:-: |:-: |:-: |:-: |
| 1.0.x | √ | √ | √ | √ | √ | √ |
| 1.1.x | not tested | √ | √ | √ | √ | √ | 
| 1.5.x | × | × | √ | √ | √ | √ |
| 1.6.x | × | × | not tested | √ | √ | √ |
| 1.7.x | × | × | × | × | √ | √ |
| 1.8.x | × | × | × | × | × | √ |

### Flink 兼容性

| Pinpoint 版本 | flink 1.3.x | flink 1.4.x |
| :-: | :-: | :-: |
| 1.7.x | √ | × |

---

# 实例

HBase 版本为 1.2.6
maven 版本为 3.5
Pinpoint 版本为 1.7.3