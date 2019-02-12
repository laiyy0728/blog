---
title: Spring Cloud 微服务（3） --- Eureka(一) <BR> 注册中心与 Eureka
date: 2019-01-17 11:41:30
updated: 2019-01-17 11:41:30
categories:
    Java
tags: 
    - SpringCloud
    - Eureka
---

Eureka 最初是针对 AWS 不提供中间服务层的负载均衡的限制开发的。 Eureka 一方面给内部服务做服务发现，另一方面可以结合 Ribbon 组件提供各种个性化的负载均衡算法。

<!-- more -->


# Spring Cloud 微服务系列版本

> Spring Boot 版本： 2.1.0.RELEASE
> Spring Cloud 版本：Finchley.RELEASE

---

# 服务发现的技术选型方案

| 名称 | 类型 | CAP | 语言 | 依赖 | 集成 | 一致性算法 |
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
|Zookeeper|General|CP|Java|JVM|Client Binding| Paxos|
|Doozer|General|CP|Go| | Client Binding | Paxos |
|Consul|General|CP|GO| | HTTP/DNS Library | Raft|
|Etcd | General| CP or Mixed(1)| Go| | Client Binding/HTTP| Raft|
|SmartStack| Dedicated|AP|Ruby|Haproxy/Zookeeper|Sidekick(nerve/synapse)||
|Eureka|Dedicated|AP|Java|JVM|Java Client||
|NSQ(lookupd)|Dedicated|AP|Go||Client Binding||
|Serf|Dedicated|AP|Go||Local CLI||
|Spotify(DNS)|15973840029|AP|N/A|Bind|DNS Library||
|SkyDNS|808376|Mixed(2)|Go||HTTP/DNS Library||

---

# Eureka 的设计理念

## 服务实例注册到服务中心

服务的注册，本质上就是在服务启动的时候，调用 Eureka Server 的 REST API 的 Registrer 方法，注册应用实例的信息。对于 Java 应用，可以使用 Netflix 的 Eureka Client 封装 API 调用；对于 Spring Cloud 应用，可以使用 `spring-cloud-starter-netflix-eureka-client` 基于 Spring Boot 自动配置，自动注册服务。

## 服务实体从服务中心剔除

正常情况下，服务实例在关闭应用的时候，应该通过钩子方法或其他生命周期回调方法，调用 Eureka Server 的 REST API 的 de-register 方法，来删除自身服务实例的信息。另外，为了解决服务挂掉，或网络中断等其他异常情况没有及时删除自身信息的问题，Eureka Server 要求 Client 定时进行续约，也就是发送心跳，来证明自己是存活的、健康的、可调用的。如果超过一定时间没有续约，则认为 Client 停掉了，Server 端会主动剔除。

## 服务实例一致性问题

由于注册中心（Eureka Server）通常来说是一个集群，就需要 Client 在集群中保持一致。Eureka 通过 `CAP`、`Peer to Peer`、`Zone + Region`、`SELF PRESERVATION` 四个方面保证一致性。


### CAP

在 [微服务与 Spring Cloud](/java/2019/01-16/spring-cloud-2.html#注册中心对比)、 [服务发现技术选型](#服务发现的技术选型方案)、[服务一致性](#服务实例一致性问题) 都提到了 `CAP`、`CP`、`AP`，那么 `CAP` 到底是个啥

- C：Consistency，数据一致性。即数据在存在多个副本的情况下，可能由于网络、机器故障、软件系统等问题早上数据写入部分副本成功，副本副本失败，进而造成副本之间数据不一致，存在冲突。满足一致性则要求对数据的更新操作成功后，多副本的数据保持一致。
- A：Availablility，服务可用性。在任何时候，客户端对集群进行读写操作时，请求能够正常响应，即，在一定的时间内完成。
- P：Partition Tolerance，分区容忍性，即发生通信故障的时候，整个集群被分隔成多个无法相互通信的分区时，集群仍然可用。

在分布式系统中，网络条件一般是不可控的，网络分区是不可避免的，因此分布式系统必须满足`分区容忍性`，所以分布式系统的设计需要在 AP、CP 之间进行选择。

**Zookeeper** 是 "C"P 的，C 之所以加引号，是因为 Zookeeper 默认并不说严格的强一致性。如：A 进行写操作，Zookeeper 在过半节点数操作成功就返回，此时如果 B 读取到的是 A 写操作没有同步到的节点，那么就读取不到 A 写入的数据。如果需要在使用的时候强一致，需要在读取的时候先执行 sync 操作，与 leader 节点先同步数据，才能保证强一致。

**Eureka** 在大规模部署的情况下，失败是不可避免的，可能会因为 Eureka 自身部署的失败，导致注册的服务不可用，或者由于网络分区，导致服务不可用。Eureka 服务注册、发现中心，保留了可用及过期的数据，以此来实现在网络分区的时候，还能正常提供服务注册发现功能。

### Peer to Peer

分布式系统的数据在多个副本直接的复制方式，可分为：主从复制、对等复制

#### 主从复制

主从复制，就是 Master-Slave 模式，有一个主副本，其他为从副本。所有的对数据的写操作，都提交到`主副本`，然后由主副本更新到其他的从副本。具体的更新方式有：同步更新、异步更新、同步异步混合更新。
对于主从复制的模式，写操作都要经过主副本，所有主副本的压力会很大。但是，从副本会分担主副本的读请求，而一个系统最大的请求都是读请求，所有可以一般情况下都是`一主多从`的架构。


#### 对等复制

即 Peer to Peer 模式。副本之间不分主从，任何副本都能接收写操作，然后每个副本之间相互进行数据更新。对于对等复制模式来说，由于任何副本都可以接收到写操作，所以不存在写操作的压力瓶颈。但是由于每个副本都能进行写操作，各个副本之间的数据同步及冲突处理是一个比较棘手的问题。

Eureka 采用的就是 `Peer to Peer`

### Zone 及 Region

Netflix 的服务大部分在 Amazon Video 上，因此 Eureka 的设计有一部分也是基于 Amazon 的 Zone、Region 基础上托管。

![Region 列表](/images/spring-cloud/eureka/region.png)

每个 Region 下，还分了几个 Zone，一个 Region 对应多个 Zone，每个 Region 之间是相互独立及隔离的，默认情况下，资源只在单个 Region 之间的 Zone 进行复制，跨 Region 之间不会进行资源复制。

![Region-Zone](/images/spring-cloud/eureka/region-zone.png)


### SELF PRESERVATION

在分布式系统中，通常需要对应用实例的存活进行健康检查，需要处理好网络抖动或短暂的不可用造成的误判；另外，Eureka Server 和 Client 之间如果存在网络分区，在极端情况下可能会导致 Eureka Server 情况部分服务的实例列表，这个将严重影响到 Eureka Server 的高可用。因此，Eureka 引入了自我保护机制(SELF PRESERVATION)。

Client 与 Server 之间有租约，Client 需要定时发送心跳维持租约，表示自己存活。Eureka 通过当前注册的实例数，计算每分钟应该从应用实例接收到的心跳数，如果最近一分钟接收到的续约次数小于指定的阈值的话，则`关闭租约失效剔除`，进制定时任务剔除失效的实例，从而保护注册信息。

---

# Eureka 入门案例

## 父 pom

```xml
<packaging>pom</packaging>

<!-- 父工程是 2.1.0.RELEASE 版本的 boot -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.1.0.RELEASE</version>
</parent>

<dependencyManagement>
    <dependencies>
        <!-- 设置全局 SpringCloud 依赖版本为 F.RELEASE -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>Finchley.RELEASE</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<!-- 设置全局依赖 SpringBoot、Web -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

## 建立 Eureka Server 子项目

***源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-eureka/spring-cloud-eureka-server-simple***

在`父pom`中建立 Eureka Server 子项目 module：`spring-cloud-eureka-server`

### pom 依赖

```xml
<!-- 父工程 -->
<parent>
    <groupId>com.laiyy.gitee</groupId>
    <artifactId>spring-cloud</artifactId>
    <version>1.0-SNAPSHOT</version>
</parent>

<dependencies>
    <!-- 引入 eureka server 依赖，注意不能少了 starter -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>
</dependencies>
```

### 启动类

```java
@SpringBootApplication
@EnableEurekaServer     // 开启 Eureka Server 
public class SpringCloudEurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringCloudEurekaServerApplication.class, args);
    }
}
```

### 配置文件

```yml
spring:
  application:
    name: eureka-server # 服务名称
server:
  port: 8761 # 服务端口

eureka:
  instance:
    hostname: localhost # host name
  client:
    fetch-registry: false # 是否获取注册表
    service-url:
      defaultZone: http://localhost:${server.port:8761}/eureka/ # 默认 Zone
    register-with-eureka: false # 是否注册自己
  server:
    enable-self-preservation: false # 是否开启自我保护，默认 true。在本机测试可以使用 false，但是在生产环境下必须为 true
```

### 验证 Eureka Server 启动结果

在浏览器输入： http://localhost:8761 ，进入 Eureka Server 可视化页面

![Eureka Server UI](/images/spring-cloud/eureka/eureka.png)


## Eureka Server UI 提示信息

在 Eureka Server 检测到异常时，会在中间以**红色加粗**字体提示信息。

在没有 Eureka Client 或 Eureka Server 检测心跳的阈值小于指定阈值，且关闭自我保护时：
> **RENEWALS ARE LESSER THAN THE THRESHOLD. THE SELF PRESERVATION MODE IS TURNED OFF.THIS MAY NOT PROTECT INSTANCE EXPIRY IN CASE OF NETWORK/OTHER PROBLEMS.**
提示说明了：1、eureka client 的心跳发送次数小于阈值(没有client，肯定小于)；2、自我保护被关闭了


在有 Eureka Client，且关闭了自我保护时：
> **THE SELF PRESERVATION MODE IS TURNED OFF.THIS MAY NOT PROTECT INSTANCE EXPIRY IN CASE OF NETWORK/OTHER PROBLEMS.**

在没有 Eureka Client 或 Eureka Server 检测心跳的阈值小于指定阈值，且开启了自我保护时：
> **EMERGENCY! EUREKA MAY BE INCORRECTLY CLAIMING INSTANCES ARE UP WHEN THEY'RE NOT. RENEWALS ARE LESSER THAN THRESHOLD AND HENCE THE INSTANCES ARE NOT BEING EXPIRED JUST TO BE SAFE.**

在有 Eureka Client 且 Eureka Server 检测心跳的阈值大于指定阈值，且开启了自我保护时，Eureka Server 会认为整个服务正常，不会有任何信息提示。


## 建立 Eureka Client 项目

***源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-eureka/spring-cloud-eureka-client-simple***

在 在`父pom`中建立 Eureka Server 子项目 module：`spring-cloud-eureka-client-simple`，建立一个简单的 eureka client 工程

### pom 

```xml
<!-- parent 都和 Eureka Server 一致，不再赘述 -->

<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
</dependencies>
```

### 启动类
```java
// EurekaClient、DiscoveryClient 本质上都是注册到服务中心的实现，EurekaClient 只针对 Eureka 使用，
// DiscoveryClient 针对不同的注册中心都可以使用。可以说 DiscoveryClient 的 EurekaClient 的一个抽象
@SpringBootApplication
@EnableDiscoveryClient //@EnableEurekaClient 
public class SpringCloudEurekaClientSimpleApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringCloudEurekaClientSimpleApplication.class, args);
    }
}
```

### 配置文件

```yml
spring:
  application:
    name: spring-cloud-eureka-client-simple # 工程名称，也是注册到 server 后的实例名称

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka      # 指定 Zone，需要与 Eureka Server 一样
  instance:
    prefer-ip-address: true # 使用 ip 注册，默认为 false。为 false 时是 机器名
    instance-id: ${spring.application.name}:${server.port} # 注册到 server 后显示的名字，默认是 机器名:name:端口

server:
  port: 8081 # 端口

```

### 在 Eureka Server 验证服务注册

![Eureka Client](/images/spring-cloud/eureka/eureka-client.png)

可以看到，eureka client 已经注册成功。 status 大致有 5 个状态：UP(正常运行)、DOWN(停机)、STATING(正在启动)、OUT_OF_SERVICE(下线)、UNKNOWN(为止)

---

# 公益 Eureka Server

如果不想在本机启动 Eureka Server，SpringCloud 中文社区提供了一个公益的 Eureka Server：[http://eureka.springcloud.cn/](http://eureka.springcloud.cn/)，在使用时只需要将 defaultZone 替换为 http://eureka.springcloud.cn/eureka 即可。

不过需要注意的是，虽然省去了启动 Eureka Server 的时间，节省了维护 Eureka Server 的资源，但是这个 Eureka Server 是一个单节点的，如果想要测试 Eureka Server 的集群高可用，还是需要在本机启动多个 Eureka Server。

公益的 Eureka Server 只能使用在本机测试中，禁止在测试环境、成产环境上使用，否则将暴露 ip、端口，造成严重的安全隐患。

