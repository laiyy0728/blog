---
title: Spring Cloud 微服务（11） --- Ribbon(一) <BR> 负载均衡与 Ribbon
date: 2019-01-25 14:57:14
updated: 2019-01-25 14:57:14
categories:
    Java
tags:
    - SpringCloud
    - Ribbon
---


通常所说的负载均衡，一般来说都是在服务器端使用 Ngnix 或 F5 做 Server 的负载均衡策略，在 Ribbon 中提到的负载均衡，一般来说是指的`客户端负载均衡`，即 ServiceA 调用 ServiceB，有多个 ServiceB 的情况下，由 ServiceA 选择调用哪个 ServiceB。

<!-- more -->

# 负载均衡与 Ribbon

负载均衡(Load Balance)，是一种利用特定方式，将流量分摊到多个操作单元上的手段，它对系统吞吐量、系统处理能力有着质的提升。最常见的负载均衡分类方式有：软负载、硬负载，对应 Ngnix、F5；集中式负载均衡、进程内负载均衡。集中式负载均衡是指位于网络和服务提供者之间，并负责把忘了请求转发到各个提供单位，代表产品有 Ngnix、F5；进程负载均衡是指从一个实例库选取一个实例进行流量导入，在微服务范畴，实例库一般是存储在 Eureka、Consul、Zookeeper 等注册中心，此时的负载均衡器类似 Ribbon 的 IPC（进程间通信）组件，因此进程内负载均衡也叫做客户端负载均衡。

Ribbon 是一个客户端负载均衡器，赋予了应用一些支配 HTTP 与 TCP 行为的能力，由此可以得知，这里的客户端负载均衡也是进程内负载均衡的一周。 Ribbon 在 SpringCloud 生态内的不可缺少的组件，没有了 Ribbon，服务就不能横向扩展。Feign、Zuul 已经集成了 Ribbon。

---

# 示例

Eureka Server 不再赘述，可以直接使用 `spring-cloud-eureka-server-simple`。

## Consumer

yml：
```yml
spring:
  application:
    name: spring-cloud-ribbon-consumer

server:
  port: 9999

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka
  instance:
    prefer-ip-address: true
    instance-id: ${spring.application.name}:${server.port}
```

配置类：
```java
@Bean
@LoadBalanced
public RestTemplate restTemplate(){
    return new RestTemplate();
}
```

`@LoadBalanced`：对 RestTemplate 启动负载均衡

Consumer Controller
```java
@RestController
public class ConsumerController {

    private final RestTemplate restTemplate;

    @Autowired
    public ConsumerController(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    @GetMapping(value = "/check")
    public String checkRibbonProvider(){
        return restTemplate.getForObject("http://spring-cloud-ribbon-provider/check", String.class);
    }

}
```

## provider

pom 依赖：
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
    </dependency>
</dependencies>
```

配置文件：
```yml
spring:
  application:
    name: spring-cloud-ribbon-provider

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka
  instance:
    prefer-ip-address: true
    instance-id: ${spring.application.name}:${server.port}
```

ProviderContr
```java
@RestController
public class ProviderController {

    @Value("${server.port}")
    private int port;

    @GetMapping(value = "/check")
    public String providerPort(){
        return "Provider Port: " + port;
    }
}
```

## 验证

分别启动 Eureka Server、Consumer、Provider，其中，Provider 以 mvn 形式启动，绑定不同的端口号：
```
mvn spring-boot:run -Dserver.port=8080
mvn spring-boot:run -Dserver.port=8081
```

postman 访问 Consumer
![第一次请求](/images/spring-cloud/ribbon/ribbon-1.png)
![第二次请求](/images/spring-cloud/ribbon/ribbon-2.png)

可以看到，Provider 两次返回值不一样，验证了负载均衡成功。

---

# 负载均衡策略

Ribbon 中提供了 `七种` 负载均衡策略

| 策略类 | 命名 | 描述 |
| :-: | :-: | :-: |
| RandomRule | 随机策略 | 随机选择 Server |
| RoundRobinRule | 轮询策略 | 按照顺序循环选择 Server |
| RetryRule | 重试策略 | 在一个配置时间段内，当选择的 Server 不成功，则一直尝试选择一个可用的 Server |
| BestAvailableRule | 最低并发策略 | 逐个考察 Server，如果 Server 的断路器被打开，则忽略，在不被忽略的 Server 中选择并发连接最低的 Server |
| AvailabilityFilteringRule | 可用过滤测试 | 过滤掉一直连接失败，并被标记未 circuit tripped（即不可用） 的 Server，过滤掉高并发的 Server |
| ResponseTimeWeightedRule | 响应时间加权策略 | 根据 Server 的响应时间分配权重，响应时间越长，权重越低，被选择到的几率就越低 |
| ZoneAvoidanceRule | 区域权衡策略 | 综合判断 Server 所在区域的性能和 Server 的可用性轮询选择 Server，并判定一个 AWS Zone 的运行性能是否可用，剔除不可用的 Zone 中的所有 Server | 


Ribbon 默认的负载均衡策略是 `轮询策略`。

## 设置负载均衡策略

### 设置全局负载均衡

创建一个声明式配置，即可实现全局负载均衡配置：

```java
@Configuration
public class RibbonConfig {
    /**
     * 全局负载均衡配置：随机策略
     */
    @Bean
    public IRule ribbonRule(){
        return new RandomRule();
    }

}
```

重启 Consumer，访问测试

## 基于注解的配置

### 空注解

声明一个空注解，用于使用注解配置 Ribbon 负载均衡

```java
public @interface RibbonAnnotation {
}
```

### 负载均衡配置类
```java
@Configuration
@RibbonAnnotation
public class RibbonAnnoConfig {

    private final IClientConfig clientConfig;

    @Autowired(required = false)
    public RibbonAnnoConfig(IClientConfig clientConfig) {
        this.clientConfig = clientConfig;
    }

    @Bean
    public IRule ribbonRule(IClientConfig clientConfig){
        return new RandomRule();
    }
}
```

### 启动类
```java
@SpringBootApplication
@EnableDiscoveryClient

@RibbonClient(name = "spring-cloud-ribbon-provider", configuration = RibbonAnnoConfig.class)
@ComponentScan(excludeFilters = {@ComponentScan.Filter(type = FilterType.ANNOTATION, value = RibbonAnnotation.class)})
public class SpringCloudRibbonConsumerApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudRibbonConsumerApplication.class, args);
    }

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate(){
        return new RestTemplate();
    }

}
```

`@RibbonClient`：针对 `spring-cloud-ribbon-provider` 服务，使用负载均衡，配置类是 `configuration` 标注的类。
`@ComponentScan`：让 Spring 不去扫描被 `@RibbonAnnotation` 类标记的配置类，因为我们的配置对单个服务生效，不能应用于全局，如果不排除，启动就会报错

如果需要对多个服务进行配置，可以使用 `@RibbonClients` 注解
```java
@RibbonClients(value = {
        @RibbonClient(name = "spring-cloud-ribbon-provider", configuration = RibbonAnnoConfig.class)        
})
```

重启 Consumer，验证基于注解的负载均衡是否成功

## 基于配置文件的负载均衡策略

语法：
```yml
{instance-id}: # instance-id 即被调用服务名称
    ribbon:
        NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule
```

---

# Ribbon 配置

## 超时与重试

HTTP 请求难免会出现请求超时，此时对调用进行时限的控制以及在时限之后的重试尤为重要。对于超时重试的配置如下：
```yml
{instance-id}: # instance-id 指的是被调用者的服务名称
  ribbon:
    ConnectTimeout: 30000 # 链接超时时间
    ReadTimeout: 30000 # 读超时时间
    MaxAutoRetries: 1 # 对第一次请求的服务的重试次数
    MaxAutoRetriesNextServer: 1 # 要重试的下一个服务的最大数量（不包括第一个服务）
    OkToRetryOnAllOperations: true # 是否对 连接超时、读超时、写超时 都进行重试
```


## Ribbon 饥饿加载

Ribbon 在进行负载均衡时，并不是启动时就加载上线文，而是在实际的请求发送时，才去请求上下文信息，获取被调用者的 ip、端口，这种方式在网络环境较差时，往往会使得第一次引起超时，导致调用失败。此时需要指定 Ribbon 客户端，进行`饥饿加载`，即：在启动时就加载好上下文。

```yml
ribbon:
  eager-load:
    enabled: true
    clients: spring-cloid-ribbon-provider
```

此时启动 consumer，会看到控制打印信息如下：
```
Client: spring-cloid-ribbon-provider instantiated a LoadBalancer: DynamicServerListLoadBalancer:{NFLoadBalancer:name=spring-cloid-ribbon-provider,current list of Servers=[],Load balancer stats=Zone stats: {},Server stats: []}ServerList:null
Using serverListUpdater PollingServerListUpdater
DynamicServerListLoadBalancer for client spring-cloid-ribbon-provider initialized: DynamicServerListLoadBalancer:{NFLoadBalancer:name=spring-cloid-ribbon-provider,current list of Servers=[],Load balancer stats=Zone stats: {},Server stats: []}ServerList:org.springframework.cloud.netflix.ribbon.eureka.DomainExtractingServerList@79e7188e
```

可以看到启动时就加载了 `spring-cloid-ribbon-provider`，并绑定了`LoadBalancer`


## Ribbon 常用配置

| 配置项 | 说明 |
| :-: | :-: |
| {instance-id}:ribbon.NFLoadBalancerClassName | 指负载均衡器类路径 |
| {instance-id}:ribbon:NFLoadBalancerRuleClassName | 指定负载均衡算法类路径 |
| {instance-id}:ribbom:NFLoadBalancerPingClassName | 指定检测服务存活的类路径 |
| {instance-id}:ribbon:NIWSServerListClassName | 指定获取服务列表的实现类路径 |
| {instance-id}:ribbon:NIWSServerListFilterClassName | 指定服务的 Filter 实现类路径 |


## Ribbon 脱离 Eureka

默认情况下，Ribbon 客户端会从 Eureka Server 读取服务注册信息列表，达到动态负载均衡的功能。如果 Eureka 是一个提供多人使用的公共注册中心(如 SpringCloud 中文社区公益 Eureka：http://eureka.springcloud.cn)，此时极易产生服务侵入问题，此时就不能从 Eureka 中读取服务列表，而应该在 Ribbon 客户端自行制定源服务地址

```yml
ribbon:
  eureka:
    enabled: false # Ribbon 脱离 Eureka 使用

{instance-id}:
  ribbon:
    listOfServers: http://localhost:8888 # 制定源服务地址
```
