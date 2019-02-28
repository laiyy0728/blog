---
title: Spring Cloud 微服务（26） --- Zuul(七) <BR> Zuul 优化
date: 2019-02-25 16:45:46
updated: 2019-02-25 16:45:46
categories:
    Java
tags: [SpringCloud, Zuul]
---


Zuul 作为一个网关中间件，需要应付各种复杂场景，整合的组件非常繁杂。在受益于其丰富的功能时，也需要面对很多问题。如：与上层负载均衡器(Nginx等)、性能、调优等。

<!-- more -->

# Zuul 应用优化

Zuul 是建立在 Servlet 上的同步阻塞架构，所有在处理逻辑上面是和线程密不可分，每一次请求都需要在线程池获取一个线程来维护 I/O 操作，路由转发的时候又需要从 http 客户端获取线程来维持连接，这样会导致一个组件占用两个线程资源的情况。所以在 Zuul 的使用中，对这部分的优化很有必要。

Zuul 的优化分为以下几个类型：
- 容器优化：内置容器 tomcat 与 undertow 的比较与参数设置
- 组件优化：内部集成的组件优化，如 Hystrix 线程隔离、Ribbon、HttpClient、OkHttp 选择等
- JVM 参数优化：适用于网关应用的 JVM 参数建议
- 内部优化：内部原生参数，内部源码，重写等


## 容器优化

把 tomcat 替换为 undertow。`undertow` 翻译为“暗流”，是一个轻量级、高性能容器。`undertow` 提供阻塞或基于 XNIO 的非阻塞机制，包大小不足 1M，内嵌模式运行时的堆内存占用只有 4M。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>
</dependency>
```

```yml
server:
  undertow:
    io-threads: 10
    worker-threads: 10
    direct-buffers: true
    buffer-size: 1024 # 字节数
```

| 配置项 | 默认值 | 说明 |
| :-: | :-: | :-: |
| server.undertow.io-threads | `Math.max(Runtime.getRuntime().availableProcessors(), 2)` | 设置 IO 线程数，它主要执行非阻塞的任务，它们负责多个连接，默认设置每个 CPU 核心有一个线程。不要设置太大，否则启动项目会报错：打开文件数过多 |
| server.undertow.worker-threads | io-threads * 8 | 阻塞任务线程数，当执行类型 Servlet 请求阻塞 IO 操作，undertow 会从这个线程池中取得线程。值设置取决于系统线程执行任务的阻塞系统，默认是 IO 线程数 * 8 |
| server.undertow.direct-buffers | 取决于 JVM 最大可用内存大小`Runtime.getRuntime().maxMemory()`，小于 64MB 默认为 false，其余默认为 true | 是否分配直接内存（NIO 直接分配的堆外内存 |
| server.undertow.buffer-size | 最大可用内存 <64MB：512 字节；64MB< 最大可用内存 <128MB：1024 字节；128MB < 最大可用内存：1024*16 - 20 字节 | 每块 buffer 的空间大小，空间越小利用越充分，设置太大会影响其他应用 |
| server.undertow.buffers-per-region | 最大可用内存 <128MB：10；128MB < 最大可用内存：20 | 每个区域分配的 buffer 数量，pool 大小是 buffer-size * buffer-per-region |

## 组件优化

### Hystrix

在 Zuul 中默认集成了 Hystrix 熔断器，使得网关应用具有弹性、容错的能力。但是如果使用默认配置，可能会遇到问题。如：第一次请求失败。这是因为第一次请求的时候，zuul 内部需要初始化很多信息，十分耗时。而 hystrix 默认超时时间是一秒，可能会不够。

解决方式：
- 加大超时时间
```yml
hystrix:
 command:
   default: 
     execution:
       isolation:
         thread:
           timeoutInMilliseconds: 5000 
```

- 禁用 hystrix 超时
```yml
hystrix:
 command:
   default: 
     execution:
       timeout:
         enabled: false    
```

Zuul 中关于 Hystrix 的配置还有一个很重要的点：`Hystrix 线程隔离策略`。

| | 线程池模式(THREAD) | 信号量模式(SEMAPHORE) |
| :-: | :-: | :-: |
| 官方推荐 | 是 | 否 |
| 线程 | 与请求线程分离 | 与请求线程公用 |
| 开销 | 上下文切换频繁，较大 | 较小 |
| 异步 | 支持 | 不支持 |
| 应对并发量 | 大 | 小 |
| 适用场景 | 外网交互 | 内网交互 |



如果应用需要与外网交互，由于网络开销比较大、请求比较耗时，选用线程隔离，可以保证有剩余容器（tomcat 等）线程可用，不会由于外部原因使线程一直在阻塞或等待状态，可以快速返回失败
如果应用不需要与外网交互，并且体量较大，使用信号量隔离，这类应用响应通常非常快，