---
title: Spring Cloud 微服务（16） --- Hystrix(四) <BR/> Hystrix 优化
date: 2019-02-11 10:02:09
updated: 2019-02-11 10:02:09
categories:
    Java
tags:
    - Hystrix
    - SpringCloud
---

Hystrix 的优化可以从`线程`、`请求缓存`、`线程传递与并发`、`命令注解`、`Collapser 请求合并` 等方面入手优化

<!-- more -->

---

# Hystrix 线程调整

线程的调整主要依赖于在生产环境中的实际情况与服务器配置进行相对应的调整，由于生产环境不可能完全一致，所以没有一个具体的值。

# 请求缓存

Hystrix 请求缓存是 Hystrix 在同一个上下文请求中缓存请求结果，与传统缓存有区别。Hystrix 的请求缓存是在同一个请求中进行，在第一次请求调用结束后对结果缓存，然后在接下来同参数的请求会使用第一次的结果。
Hystrix 请求缓存的声明周期为一次请求。传统缓存的声明周期根据时间需要设定，最长可能长达几年。

Hystrix 请求有两种方式：继承 HystrixCommand 类、使用 @HystrixCommand 注解。Hystrix 缓存同时支持这两种方案。

## Cache Consumer

***源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-hystrix/spring-cloud-hystrix-cache/spring-cloud-hystrix-cache-impl***

### pom、yml

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

```yml
server:
  port: 8989
spring:
  application:
    name: spring-cloud-hystrix-cache-impl

eureka:
  instance:
    instance-id: ${spring.application.name}:${server.port}
    prefer-ip-address: true
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

### Interceptor

```java
public class CacheContextInterceptor implements HandlerInterceptor {

    private HystrixRequestContext context;

    /**
     * 请求前
     */
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        context = HystrixRequestContext.initializeContext();
        return true;
    }

    /**
     * 请求
     */
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {

    }

    /**
     * 请求后
     */
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        context.shutdown();
    }
}


/**
 * 将 Interceptor 注册到Spring MVC 控制器
 */
@Configuration
public class CacheConfiguration {

    /**
     * 声明一个 cacheContextInterceptor 注入 Spring 容器
     */
    @Bean
    @ConditionalOnClass(Controller.class)
    public CacheContextInterceptor cacheContextInterceptor(){
        return new CacheContextInterceptor();
    }

    @Configuration
    @ConditionalOnClass(Controller.class)
    public class WebMvcConfig extends WebMvcConfigurationSupport{

        private final CacheContextInterceptor interceptor;

        @Autowired
        public WebMvcConfig(CacheContextInterceptor interceptor) {
            this.interceptor = interceptor;
        }

        /**
         * 将 cacheContextInterceptor 添加到拦截器中
         */
        @Override
        protected void addInterceptors(InterceptorRegistry registry) {
            registry.addInterceptor(interceptor);
        }
    }

}
```


### @HystrixCommand 方式
```java
// feign 调用接口
public interface IHelloService {

    String hello(int id);

    String getUserToCommandKey(@CacheKey int id);

    String updateUser(@CacheKey int id);

}


// 具体实现
@Component
public class HelloServiceImpl implements IHelloService {

    private final RestTemplate restTemplate;

    @Autowired
    public HelloServiceImpl(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    @Override
    @CacheResult
    @HystrixCommand
    public String hello(int id) {
        String result = restTemplate.getForObject("http://spring-cloud-hystrix-cache-provider-user/get-user/{1}", String.class, id);
        System.out.println("正在进行远程调用：hello " + result);
        return result;
    }

    @Override
    @CacheResult
    @HystrixCommand(commandKey = "getUser")
    public String getUserToCommandKey(int id) {
        String result = restTemplate.getForObject("http://spring-cloud-hystrix-cache-provider-user/get-user/{1}", String.class, id);
        System.out.println("正在进行远程调用：getUserToCommandKey " + result);
        return result;
    }

    @Override
    @CacheRemove(commandKey = "getUser")
    @HystrixCommand
    public String updateUser(int id) {
        System.out.println("正在进行远程调用：updateUser " + id);
        return "update success";
    }
}
```


### 继承 HystrixCommand 类形式
```java
public class HelloCommand extends HystrixCommand<String> {

    private RestTemplate restTemplate;

    private int id;

    public HelloCommand(RestTemplate restTemplate, int id){
        super(HystrixCommandGroupKey.Factory.asKey("springCloudCacheGroup"));
        this.id = id;
        this.restTemplate = restTemplate;
    }

    @Override
    protected String run() throws Exception {
        String result = restTemplate.getForObject("http://spring-cloud-hystrix-cache-provider-user/get-user/{1}", String.class, id);
        System.out.println("正在使用继承 HystrixCommand 方式进行远程调用：" + result);
        return result;
    }

    @Override
    protected String getFallback() {
        return "hello command fallback";
    }

    @Override
    protected String getCacheKey() {
        return String.valueOf(id);
    }

    public static void cleanCache(int id) {
        HystrixRequestCache.getInstance(
                HystrixCommandKey.Factory.asKey("springCloudCacheGroup"),
                HystrixConcurrencyStrategyDefault.getInstance())
                .clear(String.valueOf(id));
    }
}
```

### Controller

```java
@RestController
public class CacheController {

    private final RestTemplate restTemplate;

    private final IHelloService helloService;

    @Autowired
    public CacheController(RestTemplate restTemplate, IHelloService helloService) {
        this.restTemplate = restTemplate;
        this.helloService = helloService;
    }

    /**
     * 缓存测试
     */
    @GetMapping(value = "/get-user/{id}")
    public String getUser(@PathVariable int id) {
        helloService.hello(id);
        helloService.hello(id);
        helloService.hello(id);
        helloService.hello(id);
        return "getUser success!";
    }

    /**
     * 缓存更新
     */
    @GetMapping(value = "/get-user-id-update/{id}")
    public String getUserIdUpdate(@PathVariable int id){
        helloService.hello(id);
        helloService.hello(id);
        helloService.hello(5);
        helloService.hello(5);
        return "getUserIdUpdate success!";
    }

    /**
     * 继承 HystrixCommand 方式
     */
    @GetMapping(value = "/get-user-id-by-command/{id}")
    public String getUserIdByCommand(@PathVariable int id){
        HelloCommand helloCommand = new HelloCommand(restTemplate, id);
        helloCommand.execute();
        System.out.println("from Cache:"  + helloCommand.isResponseFromCache()) ;
        helloCommand = new HelloCommand(restTemplate, id);
        helloCommand.execute();
        System.out.println("from Cache:"  + helloCommand.isResponseFromCache()) ;
        return "getUserIdByCommand success!";
    }

    /**
     * 缓存、清除缓存
     */
    @GetMapping(value = "/get-and-update/{id}")
    public String getAndUpdateUser(@PathVariable int id){
        // 缓存数据
        helloService.getUserToCommandKey(id);
        helloService.getUserToCommandKey(id);

        // 缓存清除
        helloService.updateUser(id);

        // 再次缓存
        helloService.getUserToCommandKey(id);
        helloService.getUserToCommandKey(id);

        return "getAndUpdateUser success!";
    }
}
```

## Cache Service

***源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-hystrix/spring-cloud-hystrix-cache/spring-cloud-hystrix-cache-provider-user***

### pom、yml
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

```yml
server:
  port: 9999
spring:
  application:
    name: spring-cloud-hystrix-cache-provider-user

eureka:
  instance:
    prefer-ip-address: true
    instance-id: ${spring.application.name}:${server.port}
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

### Controller
```java
@RestController
public class UserController {

    @GetMapping(value = "/get-user/{id}")
    public User getUser(@PathVariable int id) {
        switch (id) {
            case 1:
                return new User("zhangsan", "list", 22);
            case 2:
                return new User("laiyy", "123456", 24);
            default:
                return new User("hahaha", "error", 0);
        }
    }

}
```

## 验证

### 验证 @HystrixCommand 注解形式缓存

请求 http://localhost:8989/get-user/2 ，查看控制台输出，发现控制台输出一次：
```
正在进行远程调用：hello {"username":"laiyy","password":"123456","age":24}
```

在 `HelloServiceImpl` 中，去掉 hello 方法的 @CacheResult 注解，重新启动后请求，发现控制台输出了 4 次：
```
正在进行远程调用：hello {"username":"laiyy","password":"123456","age":24}
正在进行远程调用：hello {"username":"laiyy","password":"123456","age":24}
正在进行远程调用：hello {"username":"laiyy","password":"123456","age":24}
正在进行远程调用：hello {"username":"laiyy","password":"123456","age":24}
```

由此验证 @HystrixCommand 注解形式缓存成功

### 验证 @HystrixCommand 形式中途修改参数

请求 http://localhost:8989/get-user-id-update/2 ，查看控制台，发现控制台输出：
```
正在进行远程调用：hello {"username":"laiyy","password":"123456","age":24}
正在进行远程调用：hello {"username":"hahaha","password":"error","age":0}
```

由此验证在调用 hello 方法时，hello 的参数改变后，会再次进行远程调用

### 验证清理缓存

请求 http://localhost:8989/get-and-update/2 ，查看控制，发现控制台输出：

```
正在进行远程调用：getUserToCommandKey {"username":"laiyy","password":"123456","age":24}
正在进行远程调用：updateUser 2
正在进行远程调用：getUserToCommandKey {"username":"laiyy","password":"123456","age":24}
```

修改 update 方法的 commandKey，重新启动项目，再次请求，发现控制台输出：
```
正在进行远程调用：getUserToCommandKey {"username":"laiyy","password":"123456","age":24}
正在进行远程调用：updateUser 2
```

比较后发现，修改 commandKey 后，没有进行再次调用，证明 update 没有清理掉 getUserToCommandKey 的缓存。
由此验证在调用 getUserToCommandKey 方法时，会根据 `commandKey` 进行缓存，在调用 updateUser 方法时，会根据 `commandKey` 进行缓存删除。缓存删除后再次调用，会再次调用远程接口。


## 继承 HystrixCommand 方式

访问 http://localhost:8989/get-user-id-by-command/2 ，查看控制台：

```
正在使用继承 HystrixCommand 方式进行远程调用：{"username":"laiyy","password":"123456","age":24}
from Cache:false
from Cache:true
```

可以看到，第二次请求中，`isResponseFromCache` 为 true，证明缓存生效。

由上面几种方式请求可以验证，Husyrix 的缓存可以由 @HystrixCommand 实现，也可以由继承 HystrixCommand 实现。


## 总结

- @CacheResult：使用该注解后，调用结果会被缓存，要和 @HystrixCommand 同时使用，注解参数用 cacheKeyMethod
- @CacheRemove：清除缓存，需要指定 commandKey，参数为 commandKey、cacheKeyMethod
- @CacheKey：指定请求参数，默认使用方法的所有参数作为缓存 key，直接属性为 value。
一般在读操作接口上使用 @CacheResult、在写操作接口上使用 @CacheRemove

注意事项：
再一些请求量大或者重复调用接口的情况下，可以利用缓存有效减轻请求压力，但是在使用 Hystrix 缓存时，需要注意：
- 需要开启 @EnableHystrix
- 需要初始化 HystrixRequestContext
- 在指定了 HystrixCommand 的 commandKey 后，在 @CacheRemove 也要指定 commandKey

如果不初始化 HystrixRequestContext，即在 `CacheContextInterceptor` 中不使用 `HystrixRequestContext.initializeContext()` 初始化，进行调用时会出现如下错误：
```
java.lang.IllegalStateException: Request caching is not available. Maybe you need to initialize the HystrixRequestContext?
	at com.netflix.hystrix.HystrixRequestCache.get(HystrixRequestCache.java:104) ~[hystrix-core-1.5.12.jar:1.5.12]
	at com.netflix.hystrix.AbstractCommand$7.call(AbstractCommand.java:478) ~[hystrix-core-1.5.12.jar:1.5.12]
    ...
```

另外，使用 RestTemplate 进行远程调用时，在指定远程服务时，如果出现如下错误，需要在 RestTemplate 上使用 @LoadBalance
```
java.net.UnknownHostException: spring-cloud-hystrix-cache-provider-user
	at java.net.AbstractPlainSocketImpl.connect(AbstractPlainSocketImpl.java:184) ~[na:1.8.0_171]
	at java.net.PlainSocketImpl.connect(PlainSocketImpl.java:172) ~[na:1.8.0_171]
	at java.net.SocksSocketImpl.connect(SocksSocketImpl.java:392) ~[na:1.8.0_171]
	at java.net.Socket.connect(Socket.java:589) ~[na:1.8.0_171]
	at java.net.Socket.connect(Socket.java:538) ~[na:1.8.0_171]
    ...
```