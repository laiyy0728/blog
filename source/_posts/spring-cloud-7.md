---
title: Spring Cloud 微服务 (7) --- Feign(一) <br> 远程调用、RestTemplate、Feign
date: 2019-01-22 15:48:53
updated: 2019-01-22 15:48:53
categories:
    Java
tags:
    - SpringCloud
    - Feign
---


在使用 SpringCloud 时，远程服务都是以 HTTP 接口形式对外提供服务，因此服务消费者在调用服务时，需要使用 HTTP Client 方式访问。在通常进行远程 HTTP 调用时，可以使用 RestTemplate、HttpClient、URLConnection、OkHttp 等，也可以使用 SpringCloud Feign 进行远程调用

<!-- more -->

# RestTemplate

***源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-feign/spring-cloud-feign-simple***

## 脱离 Eureka 的使用

在脱离 Eureka 使用 RestTemplate 调用远程接口时，只需要引入 web 依赖即可。

在使用 RestTemplate 时，需要先将 RestTemplate 交给 Spring 管理

```java
@Configuration
public class RestTemplateConfiguration {

    @Bean
    public RestTemplate restTemplate(){
        return new RestTemplate();
    }

}
```

编写一个 Controller，注入 RestTemplate，调用远程接口

```java
@RestController
public class RestTemplateController {

    private final RestTemplate restTemplate;

    @Autowired
    public RestTemplateController(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    @GetMapping(value = "rest-get", produces = "text/html;charset=utf-8")
    public String restTemplateGet(){
        return restTemplate.getForObject("https://gitee.com", String.class);
    }

}
```

访问 http://localhost:8080/rest-get

![RestTemplate Simple](/images/spring-cloud/feign/rest-template-simple.png)

## 关联 Eureka 使用

将服务注册到 Eureka Server，并使用 RestTemplate 调用远程 Eureka Client 服务

此时，只需要按照一个标准的 Eureka Client 编写步骤，将项目改造成一个 Eureka Client，并编写另外一个 Client。将要使用 RestTemplate 的 Client 当做服务消费者，另外一个当做服务提供者。在进行远程调用时，只需要将 `getForObject` 的 url，改为 http://service-id 即可，具体传入参数使用 `?`、`&`、`=` 拼接即可。

在注册到 Eureka Server 后，进行 RestTemplate 远程调用时，service-id 会被 Eureka Client 解析为 Server 中注册的 ip、端口，以此进行远程调用。

## Rest Template

RestTemplate 提供了 11 个独立的方法，这 11 个方法对应了各种远程调用请求

| 方法名 | http 动作 | 说明 |
| :-: | :-: | :-: |
| getForEntity() | GET | 发送 GET 请求，返回的 ResponseEntity 包含了响应体所映射成的对象 |
| getForObject() | GET | 发送 GET 请求，返回的请求体将映射为一个对象 |
| postForEntity() | POST | 发送 POST 请求，返回包含一个对象的 ResponseEntity，这个对象是从响应体中映射得到的 |
| postForObject() | POST | 发送 POST 请求，返回根据响应体匹配形成的对象 |
| postForLocation() | POST | 发送 POST 请求，返回新创建资源的 URL |
| put() | PUT | PUT 资源到指定 URL
| delete() | DELETE | 发送 DELETE 请求，执行删除操作 |
| headForHeaders() | HEAD | 发送 HEAD 请求，返回包含指定资源 URL 的 HTTP 头 |
| optionsFOrAllow() | OPTIONS | 发送 OPTIONS 请求，返回指定 URL 的 Allow 头信息 |
| execute() |  | 执行非响应 ResponseEntity 的请求 |
| exchange() | | 执行响应 ResponseEntity 的请求 |

---

# Feign

使用 RestTemplate 进行远程调用，非常方便，但是也有一个致命的问题：硬编码。 在 RestTemplate 调用中，我们每个调用远程接口的方法，都将远程接口对应的 ip、端口，或 service-id 硬编码到了 URL 中，如果远程接口的 ip、端口、service-id 有修改的话，需要将所有的调用都修改一遍，这样难免会出现漏改、错改等问题，且代码不便于维护。为了解决这个问题，Netflix 推出了 Feign 来统一管理远程调用。

## 什么是 Feign

Feign 是一个声明式的 Web Service 客户端，只需要创建一个接口，并加上对应的 Feign Client 注解，即可进行远程调用。Feign 也支持编码器、解码器，Spring Cloud Open Feign 也对 Feign 进行了增强，支持了 SpringMVC 注解，可以像 SpringMVC 一样进行远程调用。

Feign 是一种声明式、模版化的 HTTP 客户端，在 Spring Cloud 中使用 Feign，可以做到使用 HTTP 请求访问远程方法就像调用本地方法一样简单，开发者完全感知不到是在进行远程调用。

Feign 的特性：
- 可插拔的注解支持
- 可插拔的 HTTP 编码器、解码器
- 支持 Hystrix  断路器、Fallback
- 支持 Ribbon 负载均衡
- 支持 HTTP 请求、响应压缩

## 简单示例

***源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-feign/spring-cloud-feign-simple***

使用 Feign 进行 github 接口调用

### pom 依赖

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
</dependencies>
```

### 配置文件

只是进行一个简单的远程调用，不需要注册 Eureka、不需要配置文件。

### 启动类

```java
@SpringBootApplication
@EnableFeignClients
public class SpringCloudFeignSimpleApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudFeignSimpleApplication.class, args);
    }

}
```

### Feign Client 配置

```java
@Configuration
public class GiteeFeignConfiguration {
    
     /**
     * 配置 Feign 日志级别
     * <p>
     * NONE：没有日志
     * BASIC：基本日志
     * HEADERS：header
     * FULL：全部
     * <p>
     * 配置为打印全部日志，可以更方便的查看 Feign 的调用信息
     *
     * @return Feign 日志级别
     */
    @Bean
    public Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;
    }
    
}
```

### FeignClient

```java
@FeignClient(name = "gitee-client", url = "https://www.gitee.com/", configuration = GiteeFeignConfiguration.class)
public interface GiteeFeignClient {

    @RequestMapping(value = "/search", method = RequestMethod.GET)
    String searchRepo(@RequestParam("q") String query);

}
```

`@FeignClient`：声明为一个 Feign 远程调用
`name`：给远程调用起个名字
`url`：指定要调用哪个 url
`configuration`：指定配置信息

`@RequestMapping`：如同 SpringMVC 一样调用。

### Feign Controller

```java
@RestController
public class FeignController {

    private final GiteeFeignClient giteeFeignClient;

    @Autowired
    public FeignController(GiteeFeignClient giteeFeignClient) {
        this.giteeFeignClient = giteeFeignClient;
    }

    @GetMapping(value = "feign-gitee")
    public String feign(String query){
        return giteeFeignClient.searchRepo(query);
    }

}
```

### 验证调用结果

在浏览器中访问： http://localhost:8080/feign-gitee?query=spring-cloud-openfeign

![Feign To Gitee](/images/spring-cloud/feign/feign-connect-to-gitee.png)

---

# @FeignClient、@RequestMapping

## 在 Feign 中使用 MVC 注解的注意事项

在 FeignClient 中使用 `@RequestMapping` 注解调用远程接口，需要注意：

- 注解必须为 `@RequestMapping`，不能为组合注解 `@GetMapping` 等，否则解析不到
- 必须指定 method，否则会出问题
- value 必须指定被调用方的 url，不能包含域名、ip 等


## 使用 @FeignClient 的注意事项

- 在启动类上必须加上 `@FeignClients` 注解，开启扫描
- 在 FeignClient 接口上必须指定 `@FeignClient` 注解，声明是一个 Feign 远程调用

## @FeignClient

源码

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface FeignClient {
    @AliasFor("name")
    String value() default "";

    /** @deprecated */
    @Deprecated
    String serviceId() default "";

    @AliasFor("value")
    String name() default "";

    String qualifier() default "";

    String url() default "";

    boolean decode404() default false;

    Class<?>[] configuration() default {};

    Class<?> fallback() default void.class;

    Class<?> fallbackFactory() default void.class;

    String path() default "";

    boolean primary() default true;
}
```

| 字段名 | 含义 |
| :-: | :-: |
| name | 指定 FeignClient 的名称，如果使用到了 Eureka，且使用了 Ribbon 负载均衡，则 name 为被调用者的微服务名称，用于服务发现 |
| url | 一般用于调试，可以手动指定 feign 调用的地址 |
| decode404 | 当 404 时，如果该字段为 true，会调用 decoder 进行解码，否则会抛出 FeignException |
| configuration | Feign 配置类，可以自定义 Feign 的 Encoder、Decoder、LogLevel、Contract 等 |
| fallback | 容错处理类，当远程调用失败、超时时，会调用对应接口的容错逻辑。Fallback 指定的类，必须实现 @FeignClient 标记的接口 |
| fallbackFactory | 工厂类，用于生成 fallback 类的示例，可以实现每个接口通用的容错逻辑，减少重复代码 |
| path | 定义当前 FeignClient 的统一前缀 |

---

# Feign 的运行原理

- 在启动类上加上 `@EnableFeignClients` 注解，开启对 Feign Client 扫描加载
- 在启用时，会进行包扫描，扫描所有的 `@FeignClient` 的注解的类，并将这些信息注入 Spring IOC 容器，当定义的 Feign 接口中的方法被调用时，通过 JDK 的代理方式，来生成具体的 `RestTemplate`。当生成代理时，Feign 会为每个接口方法创建一个 `RestTemplate` 对象，该对象封装了 HTTP 请求需要的全部信息，如：参数名、请求方法、header等
- 然后由 `RestTemplate` 生成 Request，然后把 Request 交给 Client 处理，这里指的 Client 可以是 JDK 原生的 `URLConnection`、Apache 的 `HTTP Client`、`OkHttp`。最后 Client 被封装到 LoadBalanceClient 类，结合 Ribbon 负载均衡发起服务间的调用。

