---
title: Spring Cloud 微服务（14） --- Hystrix(二) <BR> Hystrix Dashboard
date: 2019-01-31 16:03:57
updated: 2019-01-31 16:03:57
categories:
    Java
tags:
    - SpringCloud
    - Hystrix
---

Hystrix Dashboard 仪表盘，是根据系统一段时间内发生的请求情况来展示的可视化面板，这些信息是每个 HystrixCommand 执行过程中的信息，这些信息是一个指标集合，和具体的系统运行情况。

<!-- more -->

# Hystrix Dashboard

Hystrix 的指标需要 actuator 端点进行支撑，所以需要 actuator 依赖，并公开 hsytrix.stream 端点，以便能够被顺利访问。关于 Actuator 在后面会有详细解释。

## 示例

hello-service、provider-service 是两个集成 hystrix 的微服务，服务互相调用。 
hello-service 调用 provider-service 的 get-dashboard 接口。
provider-servoce 调用 hello-service 的 hello-service 接口。

### hello-service

***源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-hystrix/spring-cloud-hystrix-dashboard/spring-cloud-hystrix-dashboard-hello-service***

#### pom

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
</dependencies>
```

#### yml

application.yml
```yml
spring:
  application:
    name: spring-cloud-hystrix-dashboard-hello-service

server:
  port: 8080
eureka:
  instance:
    prefer-ip-address: true
    instance-id: ${spring.application.name}:${server.port}
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

bootstrap.yml
```yml
management:
  endpoints:
    web:
      exposure:
        include: hystrix.stream # 开启 hystrix.stream actuator端点
feign:
  hystrix:
    enabled: true
```

#### feign、Controller、启动类

```java
// feign
@FeignClient(name = "spring-cloud-hystrix-dashboard-provider-service")
public interface ProviderService {

    @RequestMapping(value = "/get-dashboard", method = RequestMethod.GET)
    List<String> getProviderData();

}

// controller
@RestController
public class HelloController {

    private final ProviderService providerService;

    @Autowired
    public HelloController(ProviderService providerService) {
        this.providerService = providerService;
    }

    public List<String> getProviderData(){
        return providerService.getProviderData();
    }


    @GetMapping(value = "/hello-service")
    public String helloService(){
        return "hello service!";
    }
}

// 启动类
@SpringBootApplication
@EnableFeignClients
@EnableHystrix
@EnableDiscoveryClient
public class SpringCloudHystrixDashboardHelloServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudHystrixDashboardHelloServiceApplication.class, args);
    }

}
```

### provider-service

***源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-hystrix/spring-cloud-hystrix-dashboard/spring-cloud-hystrix-dashboard-provider-service***

pom、yml 配置、启动类与 hello-sercice 基本一致，需要修改 server.port、spring.application.name

#### feign、Controller

```java
// feign
@FeignClient(name = "spring-cloud-hystrix-dashboard-hello-service")
public interface ProviderService {

    @RequestMapping(value = "/hello-service", method = RequestMethod.GET)
    String helloService();

}

// Controller
@RestController
public class ProviderController {

    private final ProviderService providerService;

    @Autowired
    public ProviderController(ProviderService providerService) {
        this.providerService = providerService;
    }

    @GetMapping(value = "/get-dashboard")
    public List<String> getProviderData(){
        List<String> provider = Lists.newArrayList();
        provider.add("hystrix dashboard");
        return provider;
    }

    @GetMapping(value = "/get-hello-service")
    public String getHelloService(){
        return providerService.helloService();
    }
}
```


### hystrix-dashboard

***源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-hystrix/spring-cloud-hystrix-dashboard/spring-cloud-hystrix-dashboard-index***

#### pom
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
</dependencies>
```

#### yml

application.yml
```yml
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    instance-id: ${spring.application.name}:${server.port}
    prefer-ip-address: true
```

bootstrap.yml
```yml
spring:
  application:
    name: spring-cloud-hystrix-dashboard-index
server:
  port: 9999

management:
  endpoints:
    web:
      exposure:
        include: hystrix.stream
```

#### 启动类
```java
@SpringBootApplication
@EnableHystrixDashboard
@EnableDiscoveryClient
public class SpringCloudHystrixDashboardIndexApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudHystrixDashboardIndexApplication.class, args);
    }

}
```

`@EnableHystrixDashboard` 开启 hystrix Dashboard 页面

# Dashboard

验证 hystrix Dashboard

分别启动 hello-service、provider-sercice、hystrix-dashboard，访问 hystrix Dashboard 页面

http://localhost:9999/hystrix

![hystrix dashboard](/images/spring-cloud/hystrix/hystrix-dashboard.png)

可以看到 hystrix Dashboard 启动成功。

## Dashboard actuator

在页面上可以看到有三种监控方式

- 默认集群监控：http://turbine-hostname:port/turbine.stream 
- 指定集群监控：http://turbine-hostname:port/turbine.stream?cluster=[clusterName]
- 单个应用：http://hystrix-app:port/hystrix.stream 

其中： turbine-hostname 需要继承 Turbine。 hystrix-app 就是 instance-id

## 对 hello-service 监控

在老版本中，监控端点为：  instance-id/hystrix.stream，在新版本中，需要加上 actuator ：instance-id/actuator/hystrix.stream

hello-sercice： http://localhost:8080/actuator/hystrix.stream

![input dashboard](/images/spring-cloud/hystrix/input-dashboard.png)
![hystrix dashboard before](/images/spring-cloud/hystrix/dashboard-before.png)

可以看到，此时监控页面一直是 loading，这是因为暂时没有访问的原因。此时在 postman 多次请求 hello service 接口（localhost:8080/get-provider-data），再次查看 dashboard

![dashboard](/images/spring-cloud/hystrix/dashboard.png)

## 图形解释

- ①：访问的是哪个服务、调用的哪个方法
- ②：表示请求次数、成功数等。其中：左上角绿色代表请求成功数，左边第二个代表短路数/熔断数，右边第一个代表请求超时数，右边第二个代表线程池拒绝数，右边第三个代表失败/异常数。最右边的百分比代表最近 10 秒内的错误比例。
- ③：表示两分钟内的流量变化：流量变化越大，曲线的变化也越大；流量越大，灰色的圆圈就越大。
- ④：表示机器和集群的请求频率。host 代表当前host的请求频率；cluster 代表当前集群的请求频率
- ⑤：集群下的报告，延迟数等。hosts 代表当前集群有几个 host；median 代表请求延迟中位数；mean 请求延迟的平均数
- ⑥：最后一分钟延迟比：90th 代表 90% 的请求响应时间，99th 代表 99% 请求响应时间，99.5th 代表 99.5% 请求响应时间
- ⑦：断路器打开状态：正常情况下为 CLOED，表示断路器关闭，当一段时间内连续出现错误时，会自动变为 OPEN，代表断路器打开。
- ⑧：服务的线程状态：active：正在执行的线程，queued：队列中的请求，pool size：线程池大小，max active：最大可用线程，executions：被拒绝的线程， queue size：队列大小

## 断路器打开

关闭 provider-service 服务，多次请求 hello-service，观察 dashboard 断路器打开状态变化

![circuit-open](/images/spring-cloud/hystrix/circuit-open.png)