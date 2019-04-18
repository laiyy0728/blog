---
title: Spring Cloud 微服务（13） --- Hystrix(一) <BR>  Hystrix 介绍、Feign Hystrix 断路器
date: 2019-01-31 11:38:06
updated: 2019-01-31 11:38:06
categories:
    spring-cloud
tags:
    - SpringCloud
    - Hystrix
---


`Hystrix` 是一个延迟和容错库，目的在隔离远程系统、服务、第三方库，组织级联故障，在负载的分布式系统中实现恢复能力。
在多系统和微服务的情况下，需要一种机制来处理延迟和故障，并保护整合系统处于可用的稳定状态。`Hystrix` 就是实现这个功能的一个组件。

<!-- more -->

- 通过客户端库对延迟和故障进行保护和控制。
- 在一个复杂的分布式系统中停止级联故障
- 快速失败、迅速恢复
- 在合理的情况下回退、优雅降级
- 开启近实时监控、告警、操作控制

---

# 实例

***源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-hystrix/spring-cloud-hystrix-simple***

## pom 依赖

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
    </dependencies>
```

## 配置文件

```yml
spring:
  application:
    name: spring-cloud-hystrix-simple
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    instance-id: ${spring.application.name}:${server.port}
    prefer-ip-address: true
server:
  port: 8888
```

## Service、ServiceImpl、Controller、启动类
```java
// Service
public interface HystrixService {

    String getUser(String username) throws Exception;

}


// Service Impl
@Service
public class HystrixServiceImpl implements HystrixService {
    @Override
    @HystrixCommand(fallbackMethod = "defaultUser")
    public String getUser(String username) throws Exception {
        if ("laiyy".equals(username)) {
            return "this is real user, name:" + username;
        }
        throw new Exception();
    }

    /**
     * 在 getUser 出错时调用
     */
    public String defaultUser(String username) {
        return "this is error user, name: " + username;
    }

}


// Controller
@RestController
public class HystrixController {

    private final HystrixService hystrixService;

    @Autowired
    public HystrixController(HystrixService hystrixService) {
        this.hystrixService = hystrixService;
    }

    @GetMapping(value = "get-user")
    public String getUser(@RequestParam String username) throws Exception{
        return hystrixService.getUser(username);
    }
}


// Controller
@SpringBootApplication
@EnableDiscoveryClient
@EnableHystrix
public class SpringCloudHystrixSimpleApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudHystrixSimpleApplication.class, args);
    }

}
```

## 验证

访问 http://localhost:8888/get-user

**传入正确 username**
![正确 username](/images/spring-cloud/hystrix/hystrix-success.png)

**传入错误 username**
![错误 username](/images/spring-cloud/hystrix/hystrix-error.png)

## 注解、解释

在上例中，可以发现，在启动类上多了 `@EnableHystrix` 注解，在 Service 中多了 `@HystrixCommand(fallbackMethod = "defaultUser")` 和一个名称为 defaultUser 的方法。

- `@EnableHystrix`：开启 Hystrix 断路器
- `@HystrixCommand`：fallbackMethod 指定当该注解标记的方法出现失败、错误时，调用哪一个方法进行优雅的降级返回，对用户屏蔽错误，做优雅提示。

@HystrixCommand 需要注意
- fallbackMethod 的值，为方法名。如果方法指定的方法名不存在，会出现如下错误：
```java
com.netflix.hystrix.contrib.javanica.exception.FallbackDefinitionException: fallback method wasn't found: defaultUser1([class java.lang.String])
	at com.netflix.hystrix.contrib.javanica.utils.MethodProvider$FallbackMethodFinder.doFind(MethodProvider.java:190) ~[hystrix-javanica-1.5.12.jar:1.5.12]
	at com.netflix.hystrix.contrib.javanica.utils.MethodProvider$FallbackMethodFinder.find(MethodProvider.java:159) ~[hystrix-javanica-1.5.12.jar:1.5.12]
	at com.netflix.hystrix.contrib.javanica.utils.MethodProvider.getFallbackMethod(MethodProvider.java:73) ~[hystrix-javanica-1.5.12.jar:1.5.12]
	at com.netflix.hystrix.contrib.javanica.utils.MethodProvider.getFallbackMethod(MethodProvider.java:59) ~[hystrix-javanica-1.5.12.jar:1.5.12]
    ...
```

- fallbackMethod 方法的参数与 @HystrixCommand 标注的参数类型、顺序一致，如果不一致，会出现如下错误：
```java
com.netflix.hystrix.contrib.javanica.exception.FallbackDefinitionException: fallback method wasn't found: defaultUser([class java.lang.String])
	at com.netflix.hystrix.contrib.javanica.utils.MethodProvider$FallbackMethodFinder.doFind(MethodProvider.java:190) ~[hystrix-javanica-1.5.12.jar:1.5.12]
	at com.netflix.hystrix.contrib.javanica.utils.MethodProvider$FallbackMethodFinder.find(MethodProvider.java:159) ~[hystrix-javanica-1.5.12.jar:1.5.12]
    ...
```

---

# 断路器

***源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-hystrix/spring-cloud-hystrix-feign-broker***

断路器的作用：在服务出现错误的时候，熔断该服务，保护调用者，防止出现雪崩。

在使用 feign 断路器时，feign 默认是不开启 Hystrix 断路器的，需要手动配置。

## pom

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
    </dependency>
</dependencies>
```

## 配置文件

application.yml
```yml
feign:
  hystrix:
    enabled: true
```

boorstrap.yml
```yml
spring:
  application:
    name: spring-cloud-hystrix-feign-broker
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    instance-id: ${spring.application.name}:${server.port}
    prefer-ip-address: true
server:
  port: 8888
```

## feign、Fallback、Controller、启动类

```java
// feign
@FeignClient(name = "spring-cloud-ribbon-provider",fallback = FeignHystrixClientFallback.class)
public interface FeignHystrixClient {

    @RequestMapping(value = "/check", method = RequestMethod.GET)
    String feignHystrix();

}


// fallback
@Component
public class FeignHystrixClientFallback implements FeignHystrixClient {
    @Override
    public String feignHystrix() {
        return "error! this is feign hystrix";
    }
}

// controller
@RestController
public class FeignHystrixController {

    private final FeignHystrixClient feignHystrixClient;

    @Autowired
    public FeignHystrixController(FeignHystrixClient feignHystrixClient) {
        this.feignHystrixClient = feignHystrixClient;
    }

    @GetMapping(value = "feign-hystrix")
    public String feignHystrix(){
        return feignHystrixClient.feignHystrix();
    }
}


// 启动类
@SpringBootApplication
@EnableHystrix
@EnableDiscoveryClient
@EnableFeignClients
public class SpringCloudHystrixFeignBrokerApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudHystrixFeignBrokerApplication.class, args);
    }

}
```

## 验证

使用 `spring-cloud-ribbon-provider` 作为服务提供者，当前项目作为服务调用者

** spring-cloud-ribbon-provider 正常访问 **
![feign hystrix success](/images/spring-cloud/hystrix/feign-hystrix-success.png)

** 停止 spring-cloud-ribbon-provider 服务，再次访问 **
![feign hystrix error](/images/spring-cloud/hystrix/feign-hystrix-error.png)


可以看到，当 `spring-cloud-ribbon-provider` 服务停止时，会进入 Fallback 声明的 Class 中。

Fallback class 必须是 @FeignClient 标注的 interface 的实现类，且每个方法都需要自定义实现。
