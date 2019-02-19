---
title: Spring Cloud 微服务（23） --- Zuul(四) 限流
date: 2019-02-19 16:40:14
updated: 2019-02-19 16:40:14
categories:
    Java
tags: [SpringCloud, Zuul]
---

之前利用 Hystrix，通过熔断器实现了`通过某个阈值来对异常流量进行降级处理`。除了对异常流量进行降级之外，还可以通过 `流量排队`、`限流`、`分流`等操作，防止系统出错。

<!-- more -->

# 限流算法

限流算法一般分为 `漏桶`、`令牌桶` 两种。

## 漏桶

漏桶的圆形是一个底部有漏孔的桶，桶的上方有一个入水口，水不断流进桶内，桶下方的漏孔会以一个相对恒定的速度漏水，在`入大于出`的情况下，桶在一段时间内就会被装满，这时候多余的水就会溢出；而在`入小于出`的情况下，漏桶起不到任何作用。

当请求或者具有一定体量的数据进入系统时，在漏桶作用下，流量被整形，不能满足要求的部分被削减掉，漏桶算法能强制限定流量速度。溢出的流量可以被再次利用起来，并非完全丢弃，可以把溢出的流量收集到一个队列中，做流量排队，尽量合理利用所有资。
![leaky-bucket](/images/spring-cloud/zuul/leaky-bucket.png)

## 令牌桶

令牌桶与漏桶的区别是，桶里放的是令牌而不是流量，令牌以一个恒定的速度被加入桶内，可以积压，可以溢出。当流量涌入时，量化请求用于获取令牌，如果取到令牌则方形，同时桶内丢掉这个令牌；如果取不到令牌，则请求被丢弃。
由于桶内可以存一定量的令牌，那么就可能会解决一定程度的流量突发。这个也是漏桶与令牌桶的适用场景不同之处。

![token-bucket](/images/spring-cloud/zuul/token-bucket.png)

---

# 限流实例

在 Zuul 中实现限流，最简单的方式是使用 Filter 加上相关的限流算法，其中可能会考虑到 Zuul 多节点部署。因为算法的原因，这是需要一个 K/V 存储工具（Redis等）。

[`spring-cloud-zuul-ratelimit`](https://github.com/marcosbarbero/spring-cloud-zuul-ratelimit) 是一个针对 Zuul 的限流库
限流粒度的策略：
- user：认证用户名或匿名，针对某用户粒度进行限流
- origin：客户机 ip，针对请求客户机 ip 粒度进行限流
- url：特定 url，针对某个请求 url 粒度进行限流
- serviceId：特定服务，针对某个服务 id 粒度进行限流

限流粒度临时变量存储方式：
- IN_MEMEORY：基于本地内存，底层是 ConcurrentHashMap
- REDIS：基于 Redis K/V 存储
- CONSUL：基于 Consul K/V 存储
- JPA：基于 SpringData JPA，数据库存储
- BUKET4J：使用 Java 编写的基于令牌桶算法的限流库，四种模：`JCache`、`Hazelcast`、`Apache Ignite`、`Inifinispan`，后面三种支持异步