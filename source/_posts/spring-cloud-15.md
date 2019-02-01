---
title: Spring Cloud 微服务（15） --- Hystrix(三) --- Turbine、异常处理
date: 2019-02-01 09:53:27
updated: 2019-02-01 09:53:27
categories:
    Java
tags:
    - SpringCloud
    - Hystrix
    - Turbine
---


Hystrix Dashboard 需要输入单个服务的 hystrix.stream 监控端点，只能监控单个服务，当需要监控整个系统和集群的时候，这种方式就显得很鸡肋，此时可以使用 Turbine 来做监控。

Turbine 是为了聚合所有相关的 hystrix.stream 流的方案，然后在 hystrix dashboard 中展示。

<!-- more -->

# Turbine

微服务继续沿用上例中的 hello-service、provider-service，新建 Turbine 项目

## pom

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-turbine</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
    </dependency>
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

## 配置文件

```yml
server:
  port: 9999

spring:
  application:
    name: spring-cloud-hystrix-dashboard-turbne

eureka:
  instance:
    prefer-ip-address: true
    instance-id: ${spring.application.name}:${server.port}
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/

management:
  endpoints:
    web:
      exposure:
        exclude: hystrix.stream # Turbine 要监控的端点
turbine:
  app-config: spring-cloud-hystrix-dashboard-hello-service,spring-cloud-hystrix-dashboard-provider-service # Turbine 要监控的服务
  cluster-name-expression: "'default'" # 集群名称，默认 default
```

## 启动类

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableHystrixDashboard
@EnableTurbine // 开启 Turbine
public class SpringCloudHystrixDashboardTurbineApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudHystrixDashboardTurbineApplication.class, args);
    }

}
```

## 验证

浏览器中访问 Turbine： http://localhost:9999/hystrix ，输入 Turbine 监控端点：http://localhost:9999/turbine.stream

![turbine 没有监控到 服务](/images/spring-cloud/hystrix/turbine-no-services.png)


访问 hello-service(http://localhost:8080/get-provider-data)、provider-service(http://localhost:8081/get-hello-service)，再次查看 turbine


![turbine 监控到服务](/images/spring-cloud/hystrix/turbine-services.png)


---

# 异常处理

Hystrix 的异常处理中，有五种出错情况会被 Fallback 截获，触发 Fallbac
- FAILURE：执行失败，抛出异常
- TIMEOUT：执行超时
- SHORT_CIRCUITED：断路器打开
- THREAD_POOL_REJECTED：线程池拒绝
- SEMAPHORE_REJECTED：信号量拒绝

但是有一种类型的异常不会触发 Fallback 且不会被计数、不会熔断——`BAD_REQUEST`。BAD_ERQUEST 会抛出 HystrixBadRequestException，这种异常一般是因为对应的参数或系统参数异常引起的，对于这类异常，可以根据响应创建对应的异常进行异常封装或直接处理。

## BAD_REQUEST 处理

### pom

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
</dependencies>
```

### 配置文件

```yml
server:
  port: 9999

spring:
  application:
    name: spring-cloud-hystrix-dashboard-exception

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
    instance-id: ${spring.application.name}:${server.port}

management:
  endpoints:
    web:
      exposure:
        exclude: hystrix.stream

feign:
  hystrix:
    enabled: true
```

### 异常处理

在 Hystrix 中处理异常，需要继承 HystrixCommand<R> 并重写 run、getFallback 方法

#### bad request 

```java
public class FallBackBadRequestException extends HystrixCommand<String> {

    private static final Logger LOGGER = LoggerFactory.getLogger(FallBackBadRequestException.class);

    public FallBackBadRequestException() {
        // HystrixCommand 分组 key
        super(HystrixCommandGroupKey.Factory.asKey("GroupBadRequestException"));
    }

    @Override
    protected String run() throws Exception {
        // 直接抛出异常，模拟 BAD_REQUEST  
        throw new HystrixBadRequestException("this is HystrixBadRequestException！");
    }

    // Fallback 回调
    @Override
    protected String getFallback() {
        System.out.println("Fallback 错误信息：" + getFailedExecutionException().getMessage());
        return "this is HystrixBadRequestException Fallback method!";
    }
}
```

#### 其他错误

```java
public class FallBackOtherException extends HystrixCommand<String> {

    public FallBackOtherException() {
        super(HystrixCommandGroupKey.Factory.asKey("otherException"));
    }

    @Override
    protected String run() throws Exception {
        throw new Exception("other exception");
    }

    @Override
    protected String getFallback() {
        return "fallback!";
    }
}
```

#### 模拟 feign 调用

```java
public class ProviderServiceCommand extends HystrixCommand<String> {

    private final String name;

    public ProviderServiceCommand(String name){
        super(HystrixCommandGroupKey.Factory.asKey("springCloud"));
        this.name = name;
    }

    @Override
    protected String run() throws Exception {
        // 模拟 feign 远程调用返回
        return "spring cloud!" + name;
    }

    @Override
    protected String getFallback() {
        return "spring cloud fail!";
    }
}
```


#### Controller

```java
@RestController
public class ExceptionController {

    private static final Logger LOGGER = LoggerFactory.getLogger(ExceptionController.class);

    // 模拟 feign 调用
    @GetMapping(value = "provider-service-command")
    public String providerServiceCommand(){
        return new ProviderServiceCommand("laiyy").execute();
    }

    // bad Request
    @GetMapping(value = "fallback-bad-request")
    public String fallbackBadRequest(){
        return new FallBackBadRequestException().execute();
    }

    // 其他错误
    @GetMapping(value = "fallback-other")
    public String fallbackOther(){
        return new FallBackOtherException().execute();
    }

    // @HystrixCommand 处理 Fallback
    @GetMapping(value = "fallback-method")
    @HystrixCommand(fallbackMethod = "fallback")
    public String fallbackMethod(String id){
        throw new RuntimeException("fallback method !");
    }

    public String fallback(String id, Throwable throwable){
        LOGGER.error(">>>>>>>>>>>>>>> 进入 @HystrixCommand fallback！");
        return "this is @HystrixCommand fallback!";
    }
}
```

### 验证

依次请求 Controller 中的接口，可以看到，除了 `fallback-bad-request` 外，其他的接口都进入了 Fallback 方法中。证明了 BAD_REQUEST 不会触发 Fallback。


## BAD_REQUEST Fallback

可以使用 ErrorDecoder 对 BAD_REQUEST 进行包装

```java
@Component
public class BadRequestErrorDecoder implements ErrorDecoder {
    @Override
    public Exception decode(String methodKey, Response response) {
        if (response.status() >= 400 && response.status() <= 499) {
            String error = null;
            try {
                error = Util.toString(response.body().asReader());
            } catch (IOException e) {
                System.out.println("BadRequestErrorDecoder 出错了！" + e.getLocalizedMessage());
            }
            return new HystrixBadRequestException(error);
        }
        return FeignException.errorStatus(methodKey, response);
    }
}
```

之后在 yml 配置文件中增加对微服务调用的 ErrorDecoder 配置

