---
title: Spring Cloud 微服务（15） --- Hystrix(三) <BR/> Turbine、异常处理
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

***源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-hystrix/spring-cloud-hystrix-dashboard/spring-cloud-hystrix-dashboard-turbine***

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

***源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-hystrix/spring-cloud-hystrix-dashboard/spring-cloud-hystrix-dashboard-exception***

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

```yml
feign:
  hystrix:
    enabled: true
  client:
    config:
      spring-cloud-hystrix-dashboard-provider-service: # 针对哪个服务
        errorDecoder: com.laiyy.gitee.dashboard.springcloudhystrixdashboardexception.decoder.BadRequestErrorDecoder # 错误解码器
```

---

# Hystrix 配置

一个简单的 Hystrix 配置，基本有一下几个内容
```yml
hystrix:
  command:
    default: # default 为全局配置，如果需要针对某个 Fallback 配置，需要使用 HystrixCommandKey
      circuitBreaker:
        errorThresholdPercentage: 50 # 这是打开 Fallback 并启动 Fallback 的错误比例。默认 50%
        forceOpen: false # 是否强制打开断路器，拒绝所有请求。默认 false
      execution:
        isolation:
          strategy: THREAD # SEMAPHORE  请求隔离策略，默认 THREAD
          # 当 strategy 为 THREAD 时
          thread:
            timeoutInMilliseconds: 5000 # 执行超时时间  默认 1000
            interruptOnTimeout: true # 超时时是否中断执行，默认 true
          # 当 strategy 为 SEMAPHORE 时
          semaphore:
            maxConcurrentRequests: 10 # 最大允许请求数，默认 10
        # 是否开启超时
        timeout:
          enabled: true # 默认 true
  # 当隔离策略为 thread 时
  threadpool: 
    default: # default 为全局配置，如果需要再很对某个 线程池 配置，需要使用 HystrixThreadPoolKey
      coreSize: 10 # 默认线程池大小，默认 10
      maximumSize: 10 # 最大线程池，默认 10
      allowMaximumSizeToDivergeFromCoreSize: false # 是否允许 maximumSize 配置生效
```

## 隔离策略

```yml
hystrix:
  command:
    default: 
      execution:
        isolation:
          strategy: THREAD # SEMAPHORE  请求隔离策略，默认 THREAD
```

隔离策略有两种：线程隔离策略和信号量隔离策略。分别对应：`THREAD`、`SEMAPHORE`。

### 线程隔离
Hystrix 默认的隔离策略，通过线程池大小可以控制并发量，当线程饱和时，可以拒绝服务，防止出现问题。

优点：
- 完全隔离第三方应用，请求线程可以快速收回
- 请求线程可以继续接受新的请求，如果出现线程问题，线程池隔离是独立的，不会影响其他应用
- 当失败的应用再次变得可用时，线程池将清理并可以立即恢复
- 独立的线程池提高了并发性

缺点：
- 增加CPU开销，每个命令的执行都涉及到线程的排队、调度、上下文切换等。

### 信号量隔离

使用一个原子计数器(信号量)来记录当前有多少线程正在运行，当请求进来时，先判断计数器的数值(默认10)，如果超过设置则拒绝请求，否则正常执行，计数器+1。成功执行后，计数器-1。

与`线程隔离`的最大区别是，执行请求的线程依然是`请求线程`，而不是线程隔离中分配的线程池。

### 对单个 HystrixCommand 配置隔离策略等
```java
@HystrixCommand(fallbackMethod = "defaultUser", commandProperties = {
        // 配置线程隔离策略， value 可以是 THREAD 或 SEMAPHORE
        @HystrixProperty(name = HystrixPropertiesManager.EXECUTION_ISOLATION_STRATEGY, value = "THREAD")
})
```

### 应用场景

线程隔离：第三方应用、接口；并发量大
信号量隔离：内部应用、中间件(redis)；并发量不大

## HystrixCommandKey、HystrixThreadPoolKey

HystrixCommandKey 是一个 @HystrixCommand 注解标注的方法的 key，默认为标注方法的`方法名`，也可以使用 @HystrixCommand 进行配置
HystrixThreadPoolKey 是 Hystrix 开启线程隔离策略后，指定的线程池名称，可以使用 @HystrixCommand 配置

```java
@HystrixCommand(fallbackMethod = "defaultUser",
    // 隔离策略
    commandProperties = {
        @HystrixProperty(name = HystrixPropertiesManager.EXECUTION_ISOLATION_STRATEGY, value = "THREAD")
}, 
    // HystrixCommandKey
    commandKey = "commandKey", 
    // HystrixThreadPoolKey
    threadPoolKey = "threadPoolKey", 
    // 线程隔离策略配置超时时间
    threadPoolProperties = {
        @HystrixProperty(name = HystrixPropertiesManager.EXECUTION_ISOLATION_THREAD_INTERRUPT_ON_TIMEOUT, value = "5000")
})
```