---
title: Spring Cloud 微服务（2）                 <BR> 微服务与 Spring Cloud
date: 2019-01-16 14:32:19
updated: 2019-01-16 14:32:19
categories:
    Java
tags:
    - SpringCloud
---

应用是可以独立运行的程序代码，提供相对完善的业务功能。
架构分为：业务架构、应用架构、技术架构。业务架构决定应用架构，技术架构支撑应用架构。
架构的发展历程：`单体架构 --> 分布式架构 --> SOA 架构 --> 微服务架构`

<!-- more -->

# 微服务概述

## 单体应用架构

单体架构可以理解为一个 Java Web 工程，包含：表现成、业务层、数据访问层。从 Controller 到 Service 到 Dao，没有任何应用拆分，开发完毕后打包成一个大型的 war 部署。

![单体应用架构](/images/spring-cloud/infomation/single.png)

### 优点

* 便于发开：开发人员使用当前开发工具在短时间内就可以完成开发
* 便于测试：不需要依赖其他接口，节约时间
* 便于部署：只需将 war 部署到运行环境即可

### 缺点

* 灵活度不够：如果程序有任何修改，需要自上而下全部修改，测试的时候也不要整个程序部署完才能看到效果。在开发过程中，可能会需要等待其他开发人员开发完成才能进行自己开发的内容。
* 降低系统性能：原本可以直接访问数据库，但多出了一个 Service 层
* 启动慢：一个进程中包含了所以的业务逻辑，涉及的模块过多
* 扩展性差：增加新的功能不能针对单个点增加，要全局性的增加，牵一发动全身


## 分布式架构

按照业务垂直切分，每个应用都是单体架构，通过 API 互相调用

![分布式架构](/images/spring-cloud/infomation/api.png)

## 面向服务的 SOA 架构

SOA 架构有两个主要角色：服务的提供者（provider）、服务的消费者（consumer）。阿里的 Dubbo 的 SOA 的一个典型实现

### 优点

* 模块拆分，使用接口通信，降低了模块之间的耦合度
* 把一个大项目拆分成多个子项目，可以并行开发
* 增加功能时只需要增加一个子项目，调用其他系统接口即可
* 可以灵活分布式部署

### 缺点

* 系统之间需要远程通信，增加了开发的工作量

## 微服务架构

微服务架构是一种架构风格，对于大型复杂的业务系统，可以将业务功能拆分成多个相互独立的微服务，各个微服务之间是松耦合的，通过各种远程协议进行`同步/异步`通信，各个微服务可`单独部署`，`扩容/缩容`以及`升级/降级`。

### 微服务技术选型对比

| | Spring Cloud | Dubbo | Motan | MSEC | 其他 |
|:-:|:-:|:-:|:-:|:-:|:-:|
| 功能| 微服务完整方案| 服务治理框架|服务治理框架|服务开发运营框架|略|
| 通信方式| REST / http | RPC 协议 | RPC / Hessian2 | Protocol Buffers | gRPC、thrift |
| 服务发现 / 注册 | Eureka（AP） | ZK、Nacos | ZK / Consul | 只有服务发现 | Etcd|
| 负载均衡 | Ribbon | 客户端负载 | 客户端负载|客户端负载| Ngnix + Lua |
| 容错机制| 6 种 | 6 种 | 2 种 | 自动容错| keepalived、heartbeat|
|熔断机制| Hystrix| 无| 无| 过载保护| 无|
| 配置中心| Spring Cloud Config | Nacos | 无 | 无 | Apollo、Nacos |
| 网关 | Zuul、Gateway | 无| 无| 无| Kong、自研|
| 服务监控| Hystrix + Turbine| Dubbo + Monitor| 无| Monitor| ELK |
| 链路监控 | Sleuth + Zipkik | 无 | 无| 无| Pinpoint|
|多语言|REST 支持多语言| Java | Java | Java、C++、PHP | Java、PHP、Node.js|
|社区活跃度|高（Spring）|高（阿里）|一般|未知|略|


## 基于 Spring Cloud 的微服务解决方案

<!-- 
    markdown 语法不能合并单元格，只能用 table
    但是 markdown 不支持 <table>标签，会多出很多 <br>，解决办法：将整个 table 整合到一行
 -->

<table><thead><th style="text-align:center">组件</td><th style="text-align:center">方案1</td><th style="text-align:center">方案2</td><th style="text-align:center">方案3</td></thead><tbody><tr><td style="text-align:center">服务发现</td><td style="text-align:center">Eureka</td><td style="text-align:center">Consul</td><td style="text-align:center">etcd、Nacos</td></tr><tr><td style="text-align:center">共用组件</td><td style="text-align:center" colspan="3">服务调用：Feign、负载均衡：Ribbon、熔断器：Hystrix</td></tr><tr><td style="text-align:center">网关</td><td style="text-align:center" colspan="2">Zuul：性能低，SpringCloud Gateway：性能高</td><td style="text-align:center">自研</td></tr><tr><td style="text-align:center">配置中心</td><td style="text-align:center" colspan="3">SpringCloud Config、携程 Apollo、阿里 Nacos</td></tr><tr><td style="text-align:center">全链路监控</td><td style="text-align:center" colspan="3">Zipkin、Pinpoint、SkyWalking（推荐）</td></tr><tr><td style="text-align:center">其他</td><td style="text-align:center" colspan="3">分布式事务、Docker、gRPC、领域驱动 Halo</td></tr></tbody></table>

基于 Dubbo 的解决方案使用 `Dubbo + Nacos`。Nacos 是阿里开源的一个构建云原生应用的动态服务发现、配置、服务管理平台。

---

# Spring Cloud

## Spring Cloud 介绍

中间件：中间件是一种独立的系统软件或服务程序，分布式应用软件借助这种软件在不同的技术之间共享资源。中间件位于客户机/ 服务器的操作系统之上，管理计算机资源和网络通讯。是连接两个独立应用程序或独立系统的软件。
常见的中间件分别是：`服务治理(如 RPC)`、`配置中心`、`全链路监控`、`分布式事务`、`分布式定时任务`、`消息中间件`、`API 网关`、`分布式缓存`、`数据库中间件`等。

Spring Cloud 像是一个中间件，基于 `Spring Boot` 开发，提供一套完整的微服务解决方案。包括`服务注册与发现`、`配置中心`、`全链路监控`、`API 网关`、`熔断器` 等选型的中立的开源组件，可以随需扩展和替换组装。

## Spring Cloud 项目模块

|组件名称|所属项目|组件分类|
|:-:|:-:|:-:|
|Eureka|spring-cloud-netflix|注册中心|
|Zuul|spring-cloud-netflix|第一代网关|
|Sidecar|spring-cloud-netflix|多语言支持|
|Ribbon|spring-cloud-netflix|负载均衡|
|Hystrix|spring-cloud-netflix|熔断器|
|Turbine|spring-cloud-netflix|集群监控|
|Feign|spring-cloud-openfeign|声明式的 HTTP 客户端|
|Consul|spring-cloud-consul|注册中心|
|Gateway|spring-cloud-gateway|第二代网关|
|Sleuth|spring-cloud-sleuth|链路监控|
|Config|spring-cloud-config|配置中心|
|Bus|spring-cloud-bus|总线|
|Pipeline|spring-cloud-pipelines|部署管道|
|Dataflow|spring-cloud-dataflow|数据处理|
|Stream|spring-cloud-stream|消息驱动|

## 服务治理中间件

服务治理中间件包含：`服务注册与发现`、`服务路由`、`负载均衡`、`自我保护`、`管理机制`等。
其中，服务路由包含：`服务上下线`、`在线测试`、`机房就近选择`、`AB测试`、`灰度发布`等。
负载均衡包含：支持根据目标状态和目标权重进行负载均衡。
自我保护包含：`服务降级`、`优雅降级`、`流量控制`

## 注册中心对比

|特性|Consul|Zookeeper|etcd|Eureka|
|:-:|:-:|:-:|:-:|:-:|
|服务健康检查|服务状态、内存、硬盘等|(弱)长连接、keepalive|连接心跳|可配置支持|
|多数据中心|支持|-|-|-|
|kv 存储服务|支持|支持|支持|-|
|一致性|raft|paxos|raft|-|
|CAP|CA|CP|CP|AP|
|多语言支持|https、dns|客户端|http/grpc|http(sidecar)|
|watch支持|全量/支持long polling|支持|支持long polling|支持 long polling/大部分增量|
|自身健康|metrics|-|metrics|metrics|
|安全|acl/https|acl|https支持(弱)|-|
|Spring Cloud 集成|支持|支持|支持|支持|

## Spring Cloud 配置中心功能需求

场景支持：
> 面向全公司，充分考虑各部门需求，支持不同语言接入，充分考虑兼容性

运维集成：
> 统一数据源
> 标准化运维流程

权限管理：
> 接入权限、审核权限、下发权限...


## Spring Cloud API 网关

API 网关是出现在系统边界上的一个面向 API 的、串行集中式的强管控服务，可以理解为应用的防火墙，主要起`隔离外部访问与内部系统`的作用。主要为了解决`访问认证`、`报文转发`、`访问统计`等问题。

API 网关需要提供的功能：

> 统一接入功能：为各种服务提供统一接入服务，提供三高(高可用、高并发、高可靠)的网关服务，还需要支持负载均衡、容灾切换、异地多活
> 协议适配：对外提供 HTTP、HTTPS 访问，对内提供各种协议，如：HTTP、HTTPS、RPC、REST 等
> 流量监控：当大流量请求时，进行`流量监控`、`流量挑拨`；当后端出现异常时，需要进行`服务熔断`、`服务降级`；在异地多活中，需要进行`流量分片`等。
> 网关防护：所有请求进行安全过滤，对 IP 黑名单、URL 黑名单封禁、风控防刷、防恶意攻击等。

Zuul、Gateway 的比较
||Zuul|Gateway|
|:-:|:-:|:-:|
|实现难度|低|高|
|底层|http、https、rest|Netty|
|灰度、降级、标签路由、限流、WAF封禁|需要自定义 Filter|配置|
|安全、监控/埋点|自定义 Intercepter|自定义配置|


## Spring Cloud 全链路监控

全链路监控需要包含的功能：
> 定位慢调用：慢 web 服务、慢 rest/rpc 服务、慢 SQL
> 定位错误：4XX、5XX、Service Error
> 定位异常：Error Exception、Fatal Exception
> 展现依赖和拓扑：域拓扑、服务拓扑、trace 拓扑
> trace 调用链：将端到端的调用以及附加的上下文信息、异常日志信息、每个调用点的耗时都进行展示
> 应用警告：根据设置的警告规则，扫描指标数据，并进行信息上报


