---
title: Spring Cloud 微服务（4） --- Eureka(二) <br>  REST API、核心类、调优
date: 2019-01-18 16:13:34
updated: 2019-01-18 16:13:34
categories:
    spring-cloud
tags:
    - SpringCloud
    - Eureka
---

在上一篇已经了解到了`服务注册与发现`、`Eureka`、`Eureka 简单示例`、`Eureka Server 中查看 Client 状态`等。接下来需要了解 Eureka 的 `REST API`、`核心类`、`调优`等惭怍。

<!-- more -->

# REST API

在 Eureka Server 的可视化页面中，我们可以看到每个微服务的注册信息。在 Server、Client 的配置文件中，都指定了一个 `defaultZone: ip:port/eureka/`，那么这个配置的作用是什么？为什么在 ip、端口 后面要加上一个 /eureka ？

`/eureka` 就是 Eureka 的 REST API 的端点地址。

## /eureka/ 端点

启动 Eureka Server、Client，在浏览器中输入： http://localhost:8761/eureka/apps ，可以看到如下信息：

```xml
<!-- 所有工程 -->
<applications> 
    <versions__delta>1</versions__delta>   
    <apps__hashcode>UP_1_</apps__hashcode>    
    <application> 
        <!-- 单个实例的名称 -->
        <name>SPRING-CLOUD-EUREKA-CLIENT-SIMPLE</name>    
         <instance> 
            <!-- 单个实例的 instance-id -->
            <instanceId>spring-cloud-eureka-client-simple:8081</instanceId>    
            <!-- 实例的 hostname，没有指定时用 ip -->
            <hostName>10.10.10.141</hostName>    
            <app>SPRING-CLOUD-EUREKA-CLIENT-SIMPLE</app>    
            <!-- 实例的ip -->
            <ipAddr>10.10.10.141</ipAddr>    
            <!-- 实例状态 -->
            <status>UP</status>    
            <overriddenstatus>UNKNOWN</overriddenstatus>    
            <!-- 端口 -->
            <port enabled="true">8081</port>    
            <!-- https -->
            <securePort enabled="false">443</securePort>    
            <countryId>1</countryId>    
            <dataCenterInfo class="com.netflix.appinfo.InstanceInfo$DefaultDataCenterInfo"> 
                <name>MyOwn</name> 
            </dataCenterInfo>    
            <leaseInfo> 
                <!-- 每多长时间续约一次，单位 秒 -->
                <renewalIntervalInSecs>30</renewalIntervalInSecs>    
                <!-- 续约过期时间，单位秒。规定时间内没有续约会剔除 Eureka Server -->
                <durationInSecs>90</durationInSecs>    
                <!-- 注册时间 -->
                <registrationTimestamp>1547800471059</registrationTimestamp>    
                <!-- 上一次续约时间 -->
                <lastRenewalTimestamp>1547800471059</lastRenewalTimestamp>    
                <evictionTimestamp>0</evictionTimestamp>    
                <serviceUpTimestamp>1547800471059</serviceUpTimestamp> 
            </leaseInfo>    
            <metadata> 
                <management.port>8081</management.port>    
                <jmx.port>10235</jmx.port> 
            </metadata>    
            <!-- 主页面 -->
            <homePageUrl>http://10.10.10.141:8081/</homePageUrl>    
            <!-- 实例信息 -->
            <statusPageUrl>http://10.10.10.141:8081/actuator/info</statusPageUrl>    
            <!-- 实例健康检查 -->
            <healthCheckUrl>http://10.10.10.141:8081/actuator/health</healthCheckUrl>    
            <vipAddress>spring-cloud-eureka-client-simple</vipAddress>    
            <secureVipAddress>spring-cloud-eureka-client-simple</secureVipAddress>    
            <isCoordinatingDiscoveryServer>false</isCoordinatingDiscoveryServer>    
            <lastUpdatedTimestamp>1547800471059</lastUpdatedTimestamp>    
            <lastDirtyTimestamp>1547800470997</lastDirtyTimestamp>    
            <actionType>ADDED</actionType> 
        </instance> 
    </application> 
</applications>
```

此时看到的信息，是在 Eureka Server 中注册的所有Client 的信息。如果想要查询单个 Client 的信息，可以访问 http://localhost:8761/eureka/apps/{application.name} ，如：http://localhost:8761/eureka/apps/SPRING-CLOUD-EUREKA-CLIENT-SIMPLE


## 常用的 REST API
常用的 Eureka REST API 除了 /eureka/apps 之外，还有如下接口

| 操作 | http 动作 | 接口 | 描述 |
| :-: | :-: | :-: | :-: |
| 注册新的应用实例 | POST | /eureka/apps/{appId} | 可以输入 json 或者 xml 格式的 body，成功返回 204 |
| 注销实例 | DELETE | /eureka/apps/{appId}/{instanceId} | 成功返回 200 |
| 发送心跳 | PUT | /eureka/apps/{appId}/{instanceId} | 成功返回 200，instanceId 不存在返回 404 |
| 查询所有实例 | GET | /eureka/apps | 成功返回 200，输出 json 或 xml 格式的 body |
| 查询单个实例 | GET | /eureka/apps/{appId} | 成功返回 200，输出json 或 xml 格式的 body |
| 根据 appId、instanceId 查询 | GET | /eureka/apps/{appId}/{instanceId} | 成功返回 200，输出 json 或 xml 格式的 body |
| 暂停某个实例 | PUT | /eureka/apps/{appId}/{instanceId}/status?value=OUT_OF_SERVICE | 成功返回 200，失败返回 500 |
| 恢复某个实例 | DELETE | /eureka/apps/{appId}/{instanceId}/status?vlaue=UP(value 可不传) | 成功返回 200，失败返回 500 |
| 更新元数据 | PUT | /euerka/apps/{appId}/{instanceId}/metadata?key=value | 成功返回 200，失败返回 500 |
| 根据虚拟 ip 查询 | GET | /eureka/vip/{vipAddr} | 成功返回 200，输出 json 或 xml 格式的 body |
| 根据基于 htpps 的虚拟 ip 查询 | GET | /eureka/svip/{svipAddr} | 成功返回 200，输出 json 或 xml 格式的 body |

在进行新实例的注册时，传入的 json、xml 的格式需要与调用`获取单个实例`所返回的数据格式一致。具体的数据需要自己指定，需要注意的是，如果要注册的实例的ip、端口已经存在的话，不会再次注册，需要修改 instanceId。

---

# Eureka 核心类

Eureka 提供了一些核心的类，这些类中保存了 Eureka Server、Client 的注册信息、运行时的信息等。

## InstanceInfo

InstanceInfo 代表了注册的服务实例(位置： com/netflix/appinfo/InstanceInfo.java)

| 字段 | 说明 |
| :-: | :-: |
| app | 应用名称 |
| appGroupName | 应用所属群组 |
| ipAddr | ip 地址 |
| sid | 已废弃，默认 na |
| port | 端口号 |
| securePort | https 端口 |
| homePageUrl | 应用实例的首页 url |
| statusPageUrl | 应用实例的状态页 url |
| healthPageUrl | 应用实例的健康检查 url |
| secureHealthPageUrl | 应用实例的健康检查 https url |
| vipAddress | 虚拟 ip 地址 |
| secureVipAddress | https 的虚拟 ip 地址 |
| countryId | 已废弃，默认 1，代表 US |
| dataCenterI | dataCenter 信息，Netflix、Amazon、MyOwen |
| hostName | 主机名称（默认 ip |
| status | 状态，如：UP、DOWN、STATING、OUT_OF_SERVICE、UNKNOWN |
| overrideenstatus | 外界需要强制覆盖的状态，默认为 UNKNOWN |
| leaseInfo | 租约信息 |
| isCorrdinatindDiscoveryServer | 首先标示是否是 DiscoveryServer，其次标示该 DiscoveryServer 是否是响应你请求的实例 |
| metadata | 应用实例的元数据信息 |
| lastUpdateTimestamp | 状态信息最后更新时间 |
| lastDirtyTimestamp | 实例信息的最新过期时间，在 Client 端用于标志该实例信息是否与 Eureka Server 一致，在 Server 端则与多个 Server 之间进行信息同步 |
| actionType | 标示 Server 对该实例进行的操作，包括：ADDED、MODIFIED、DELETED |
| args | 在 AWS 的 autoscaling group 名称 |
可以看出，整个 InstanceInfo 的返回值信息，就是访问 `/eureka/apps/{appId}` 的返回值，也是通过 REST API 向 Eureka Server 注册时的 body。

## LeaseInfo

LeaseInfo 标识应用实例的租约信息(位置： com/netflix/appinfo/LeaseInfo.java)

| 字段 | 说明 |
| :-: | :-: |
| renewalIntervalInSecs | Client 续约时间间隔(秒) |
| durationInSecs | Client 的租约有效时间长(秒) |
| registreationTimestamp | 第一次注册时间(毫秒时间戳) |
| laseRenewalTimestamp | 最后一次续约时间(毫秒时间戳) |
| evicationTimestamp | 租约被剔除时间(毫秒时间戳) |
| serviceUpTimestamp | Client 被标记为 UP 状态的时间(毫秒时间戳) |

## ServiceInstanceInfo 

ServiceInstanceInfo 是一个标识一个应用实例的接口，约定了服务的发现的实例应用有哪些通用信息(位置： org/springframework/cloud/client/ServiceInstanceInfo.java)

| 方法 | 说明 |
| :-: | :-: |
| getSercieId() | 获取服务 id |
| getHost() | 获取实例的 HOST |
| getPort() | 获取实例的端口 |
| isSecure() | 是否开启 https |
| getUri() | 实例的 uri 地址 |
| getMetadata() | 实例的元数据信息 |
| getScheme() | 实例的 scheme |

对于 ServiceInstanceInfo 接口的实现为：EurekaRegistration(位置：org/springframework/cloud/netflix/eureka/serviceregistry/EurekaRegistration.java)，EurekaRegistration 同时还实现了 Closeable 接口，这个接口的作用之一是在 close 的时候调用 eurekaClient.shutdown() 方法，实现优雅关闭 Eureka Client。

## InstanceStatus

InstanceStatus 是一个枚举类型，源码如下：
```java
public static enum InstanceStatus {
    UP,
    DOWN,
    STARTING,
    OUT_OF_SERVICE,
    UNKNOWN;

    private InstanceStatus() {
    }

    public static InstanceInfo.InstanceStatus toEnum(String s) {
        if (s != null) {
            try {
                return valueOf(s.toUpperCase());
            } catch (IllegalArgumentException var2) {
                InstanceInfo.logger.debug("illegal argument supplied to InstanceStatus.valueOf: {}, defaulting to {}", s, UNKNOWN);
            }
        }

        return UNKNOWN;
    }
}
```

其中定义了 5 种状态，对应 Client 的 5 种状态。

---

# 服务的核心操作

对于服务发现来说，一般都是围绕几个核心的概念进行设计：
> 服务发现（register）
> 服务下线（cancel）
> 服务租约（renew）
> 服务剔除（evict）

围绕这几个概念，Eureka 设计了一些核心的操作类：
- com/netflix/eureka/lease/LeaseManager.java
- com/netflix/discovery/shared/LookupService.java
- com/netflix/eureka/registry/InstanceRegistry.java
- com/netflix/eureka/registry/AbstractInstanceRegistry.java
- com/netflix/eureka/registry/PeerAwareInstanceRegistry.java

在 Netflix Eureka 的基础上，Spring Cloud 抽象或定义了几个核心类:
- org/springframewor/cloud/netflix/eureka/server/InstanceRegistry.java
- org/springframewor/cloud/client/serviceregistry/ServiceRegistry.kava
- org/springframewor/cloud/netflix/eureka/serviceregistry/EurekaServiceRegistry.java
- org/springframewor/cloud/netflix/eureka/serviceregistry/EurekaRegistration.java
- org/springframewor/cloud/netflix/eureka/EurekaClientAutoConfiguration.java
- org/springframewor/cloud/netflix/eureka/EurekaClientConfigBean.java
- org/springframewor/cloud/netflix/eureka/EurekaInstanceConfigBean.java

其中：`LeaseManager`、`LookupService` 是 Eureka 关于服务发现相关操作操作定义的接口类，`LeaseManager` 定义了服务写操作相关方法，`LookupService` 定义查询操作的相关方法。

## LeaseManager

```java
public interface LeaseManager<T> {
    
    void register(T r, int leaseDuration, boolean isReplication);

    boolean cancel(String appName, String id, boolean isReplication);

    boolean renew(String appName, String id, boolean isReplication);

    void evict();
}、
```

- register：用于注册服务实例信息
- cancel：用于删除服务实例信息
- renew：用于和 Eureka Server 进行心跳操作，维持租约
- evict：Server 端的方法，用于剔除租约过期的服务实例。


## LoopupService

```java
public interface LookupService<T> {

    Application getApplication(String appName);

    Applications getApplications();

    List<InstanceInfo> getInstancesById(String id);

    InstanceInfo getNextServerFromEureka(String virtualHostname, boolean secure);
}
```
- getApplication：根据 appName 获取服务信息
- getApplications：获取所有注册的服务信息
- getInstancesById：根据 appid，获取所有实例
- getNextServerFromEureka：根据虚拟hostname、是否是 https，获取下一个服务实例方法（默认轮训获取）

---


# Eureka 参数调优

## Client 端

### 基本参数

| 参数 | 默认值 | 说明 |
| :-: | :-: | :-: |
| eureka.client.avaliability-zones | | 告知 Client 有哪些 regin 和 zone，支持配置修改运行时生效 |
| eureka.client.filter-only-up-instances | true | 是否滤出 InstanceStatus 为 UP 的实例 |
| eureka.clint.region | us-east-1 | 指定 region，当 datacenters 为 AWS 时适用 |
| eureka.client.register-with-eureka | true | 是否将实例注册到 Eureka Server |
| eureka.client.prefer-same-zone-eureka | true | 是否优先使用和该应用实例处于相同 Zone 的 Eureka Server |
| eureka.client.on-demand-update-status-change | trye | 是否将本地实例状态的更新，通过 ApplicationInfoManager 实时同步到 Eureka Server(这个同步请求有流量限制) |
| eureka.instance.matadata-map | | 指定实例的元数据信息 |
| eureka.instance.prefer-ip-address | false | 是否优先使用 ip 地址来代替 hostname 作为实例的 hostname 字段值 |
| eureka.instance.lease-exporation-duration-in-seconds | 90 | 指定 Eureka Client 间隔多久向 Server 发送心跳 |

### 定时任务参数

| 参数 | 默认值(时间单位：秒，非时间单位：个) | 说明 |
| :-: | :-: | :-: |
| eureka.client.cache-refresh-executor-thread-pool-size | 2 | 刷新缓存的 CacheRefreshThread 线程池大小 |
| eureka.client.cache-refresh-executor-exponential-back-off-bound | 10 | 调度任务执行时，下次调度的延迟时间 |
| eureka.client.heartbeat-executor-thread-pool-size | 2 | 执行心跳 HeartbeatThread 的线程池大小 |
| eureka.client.heartbeat-executor-exponential-back-off-bound | 10 | 调度任务执行时，下次调度的延迟时间 |
| eureka.client.registry-fetch-interval-seconds | 30 | CachaRefreshThread 线程调度频率 |
| eureka.client.eureka-service-url-poll-interval-seconds | 5*60 | AsyncResolver.updateTask 刷新 Eureka Server 地址的时间间隔 |
| eureka.client.initial-instance-info-replication-interval-seconds | 40 | InstanceInfoReplicator 将实例信息变更同步到 Eureka Server 的初始延时时间 |
| eureka.client.instance-infi-replication-interval-seconds | 30 | InstanceInfiReplicator 将实例信息变更同步到 Eureka Server 的时间间隔 |
| eureka.client.lease-renewal-interval-in-seconds | 30 | Eureka Client 向 Eureka Server 发送心跳的时间间隔 |

### http 参数

Eureka Client 底层使用 HttpClient 与 Eureka Server 通信。

| 参数 | 默认值 | 说明 |
| :-: | :-: | :-: |
| eureka.client.eureka-server-connect-timeout-seconds | 5 | 连接超时时间 |
| eureka.client.eureka-server-read-timeout-seconds | 8 | 读超时时间 |
| eureka.client.eureka-server-total-connections | 200 | 连接池最大连接数 |
| eureka.client.eureka-server-total-connections-per-host | 50 | 每个 host 能使用的最大链接数 |
| eureka.client.eureka-connection-idle-timeout-seconds | 30 | 连接池空闲连接时间 |


## Server 端

Server 端的参数调优分为：基本参数，Response Cache、Peer、Http 等

### 基本参数

| 参数 | 默认值 | 说明 | 
| :-: | :-: | :-: |
| eureka.server.enable-self-preservation | true | 是否开启自我保护模式 |
| eureka.server.renewal-percent-threshold | 0.85 | 每分钟需要收到的续约次数阈值(心跳数/client实例数) |
| eureka.instance.registry.expected-number-of-renews-per-min | 1 | 指定每分钟需要收到的续约次数，实际上，在源码中被写死为 count * 2 |
| eureka.server-renrewal-threshold-update-interval-ms | 15 分钟 | 指定 updateRenewalThreshold 定时任务的调度频率，动态更新 expectedNumberOfRenewsMin 以及 numberOfNewsPerMinThreshold 的值 |
| eureka.server.evication-interval-timer-in-ms | 60*1000 | 指定 EvicationTask 定时任务调度频率，用于剔除过期的实例 |

### Response Cache 参数

Eureka Server 为了提升自身 REST API 接口的性能，提供了两个缓存：一个是基于 `ContioncurrentMap` 的 `readOnlyCacheMap`，一个是基于 `Guava Chahe` 的 `readWriteCacheMap`。

| 参数 | 默认值 | 说明 |
| :-: | :-: | :-: |
| eureka.server.use-read-only-response-cache | true | 是否使用只读的 Response Cache |
| eureka.server.response-cache-update-interval-ms | 30 * 1000 | 设置 CacheUpdateTask 的调度时间间隔，用于从 readWriteCacheMap 更新数据到 readOnlyCacheMap。仅在 `eureka.server.use-read-only-response-cache` 为 true 时生效 |
| eureka.server.response-cache-auto-expiration-in-seconds |180 | 设置 `readWriteCacheMap` 的过期时间 |


### peer 参数

| 参数 | 默认值 | 说明 |
| :-: | :-: | :-: |
| eureka.server.peer.eureka-nodes-update-interval-ms | 10分钟 | 指定 peersUpdateTask 调度的时间间隔，用于配置文件刷新 `peerEurekaNodes` 节点的配置信息 |
| eureka.server.peer-eureka-status-refresh-time-interval-ms | 30*1000 | 指定更新 Peer nodes 状态的时间间隔 |

### http 参数

| 参数 | 默认值 | 说明 |
| :-: | :-: | :-: |
| eureka.server.peer-node-connect-timeout-ms | 200 | 链接超时时间 |
| eureka.server.peer-node-read-timeout-ms | 200 | 读超时时间 |
| eureka.server.peer-node-total-connections | 1000 | 连接池最大连接数 |
| eureka.server.peer-node-total-connections-per-host | 500 | 每个 host 能使用的最大连接数 |
| eureka.server.peer-node-connection-idle-timeout-seconds | 30 | 连接池中链接的空闲时间 |

