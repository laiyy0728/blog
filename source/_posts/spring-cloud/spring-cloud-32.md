---
title: Spring Cloud 微服务（32） --- APM(一) <BR> Sleuth
date: 2019-04-10 21:19:14
updated: 2019-04-10 21:19:14
categories: spring-cloud
tags: [SpringCloud, APM]
---

在微服务架构下，服务按照不同的纬度进行拆分，一次请求可能会涉及到多个服务，并且有可能是由不同的团队开发，可能使用不同的编程语言实现，有可能部署在几百台、几千台服务器上，横跨多个不同的数据中心。因此，需要一些可以帮助理解系统行为、分析性能问题的工具，以便在发生故障时，快速定位、解决问题。此类工具称为 `APM`

<!-- more -->

# APM

最出名的 `APM` 是谷歌公开的论文中提到的 `Dapper`。Dapper 对分布式跟踪系统提出了如下需求：
- 性能低损耗：分布式跟踪系统对服务的性能损耗应尽可能做到可以忽略不计，尤其是对性能敏感的应用不能产生损耗。
- 对应用透明：尽可能使用非侵入的方式实现跟踪，尽可能做到业务代码的低侵入，对业务开发人员做到透明化。
- 可伸缩性：是指不能随着微服务和集群规模的扩大而使用分布式跟踪系统瘫痪。
- 跟踪数据可视化、迅速反馈：要有可视化的监控界面，从跟踪数据收集、处理、到结果的展现，尽量做到快速，这样可以对系统的异常状况作出快速反应。
- 持续监控：要求分布式跟踪系统必须是 7X24 小时工作，否则很难定位到系统偶尔抖动的行为。

在 APM 中的一些术语
- Span：基本工作单元。如：发送一次 RPC 请求，就是一个新的 Span。Span 通过一个 64 位的 ID 标识，还包含有描述、事件时间戳、标签、调用它的 Span 的 ID、处理器 ID（一般为 ip 地址）。注意：第一个 Span 是 root Span，它的 ID 和 Trace 的 ID 一样
- Trace：一系列 Span 组成的树状结构，简单的说就是一次调用请求
- Annotation：标注，用来描述事件的实时状态。有如下状态
> cs：Client Sent。客户端发起请求，表示一个 Span 开始
> sr：Server Received。服务方接收到请求，并开始处理，其值减去 cs 时间，就是网络延迟时间
> ss：Server Sent。表示请求处理完成，将响应数据返回给客户端。其值减去 sr 时间，就是服务方处理时间
> cr：Client Received。客户端接收到服务方的返回值，是当前 Span 结束的信号。其值减去 cs，就是一次请求的完整处理时间。

---

# Sleuth

`Sleuth` 是 SpringCloud 的分布式跟踪系统，通过 Trace 定义一次业务调用链，根据它的信息，我们能知道有多少系统参与了该业务处理。而系统间的调用顺序、时间戳信息，通过 Span 记录。Trace 和 Span 整合，就能知道该业务的完整调用链。

## 一个简单的 Sleuth

***源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-apm/spring-cloud-apm-sleuth***

### 通用 pom
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-sleuth</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
</dependencies>
```

### Provider

```yml
server:
  port: 8082
spring:
  application:
    name: spring-cloud-apm-sleuth-provider
```

```java
@SpringBootApplication
@RestController
public class SpringCloudApmSleuthProviderApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudApmSleuthProviderApplication.class, args);
    }

    private static final Logger LOGGER = LoggerFactory.getLogger(SpringCloudApmSleuthProviderApplication.class);

    @GetMapping(value = "/say")
    public String hello(String name){
        LOGGER.info(">>>>>>>>>>>>>>>>>>> 接收到参数：{}<<<<<<<<<<<<<<<<<<<<", name);
        String result = "你好啊~" + name;
        LOGGER.info(">>>>>>>>>>>>>>>>>>> 返回值：{} <<<<<<<<<<<<<<<<<<<<<<", result);
        return result;
    }

}
```

### Consumer

```yml
server:
  port: 8081
spring:
  application:
    name: spring-cloud-apm-aleuth-consumer
```

**Feign**
```java
@FeignClient(name = "spring-cloud-apm-sleuth-provider", url = "localhost:8082")
public interface HelloService {

    @RequestMapping(value = "/say")
    String sayHello(@RequestParam("name") String name);

}
```

**configuration**
```java
@Configuration
public class ConsumerConfiguration {

    private final BeanFactory beanFactory;

    @Autowired
    public ConsumerConfiguration(BeanFactory beanFactory) {
        this.beanFactory = beanFactory;
    }

    @Bean
    public RestTemplate restTemplate(){
        return new RestTemplate();
    }

    @Bean
    public ExecutorService executorService(){
        ExecutorService executorService = Executors.newFixedThreadPool(2);
        return new TraceableExecutorService(this.beanFactory, executorService);
    }

}
```

**Controller**
```java
@RestController
public class ConsumerController {

    private static final Logger LOGGER = LoggerFactory.getLogger(ConsumerController.class);

    private final HelloService helloService;
    private final RestTemplate restTemplate;
    private final ExecutorService executorService;

    @Autowired
    public ConsumerController(HelloService helloService, RestTemplate restTemplate, ExecutorService executorService) {
        this.helloService = helloService;
        this.restTemplate = restTemplate;
        this.executorService = executorService;
    }

    @GetMapping(value = "/hello-feign")
    public String helloByFeign(String name){
        LOGGER.info(">>>>>>>>>>>>>>>>>> feign 调用，参数：{} <<<<<<<<<<<<<<<<<<<<", name);
        String result = helloService.sayHello(name);
        LOGGER.info(">>>>>>>>>>>>>>>>>> feign 调用，结果：{} <<<<<<<<<<<<<<<<<<<<", result);
        return result;
    }

    @GetMapping(value = "/hello-rest")
    public String helloByRest(String name){
        LOGGER.info(">>>>>>>>>>>>>>>>>> rest 调用，参数：{} <<<<<<<<<<<<<<<<<<<<", name);
        String url = "http://localhost:8082/say?name=" + name;
        String result = restTemplate.getForObject(url, String.class);
        LOGGER.info(">>>>>>>>>>>>>>>>>> rest 调用，结果：{} <<<<<<<<<<<<<<<<<<<<", result);
        return result;
    }

    @GetMapping(value = "/hello-thread")
    public String helloByThread(String name) throws ExecutionException, InterruptedException {
        LOGGER.info(">>>>>>>>>>>>>>>>>> 线程 调用，参数：{} <<<<<<<<<<<<<<<<<<<<", name);
        String url = "http://localhost:8082/say?name=" + name;
        Future<String> future = executorService.submit(() -> {
            LOGGER.info(">>>>>>>>>>>>> 进入线程，参数：{} <<<<<<<<<<<<<<<<<<<", name);
            return restTemplate.getForObject(url, String.class);
        });
        String result = future.get();
        LOGGER.info(">>>>>>>>>>>>>>>>>> 线程 调用，结果：{} <<<<<<<<<<<<<<<<<<<<", result);
        return result;
    }

}
```

**启动类**
```java
@SpringBootApplication
@EnableFeignClients
public class SpringCloudApmSleuthConsumerApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudApmSleuthConsumerApplication.class, args);
    }

}
```

## 验证

### Fegin 访问验证

请求 http://localhost:8081/hello-feign?name=张三 ，Consumer 控制台打印信息如下：

![feign 调用 consumer 打印](/images/spring-cloud/apm/sleuth-feign-consumer.png)

Privider 打印信息如下：
![feign 调用 provider 打印](/images/spring-cloud/apm/sleuth-feign-provider.png)

### RestTemplate 验证

请求 http://localhost:8081/hello-rest?name=张三 ，Consumer 控制台打印信息如下：

![rest 调用 consumer 打印](/images/spring-cloud/apm/sleuth-rest-consumer.png)

Privider 打印信息如下：
![rest 调用 provider 打印](/images/spring-cloud/apm/sleuth-rest-provider.png)


### 线程验证


请求 http://localhost:8081/hello-thread?name=张三 ，Consumer 控制台打印信息如下：

![thread 调用 consumer 打印](/images/spring-cloud/apm/sleuth-thread-consumer.png)

Privider 打印信息如下：
![thread 调用 provider 打印](/images/spring-cloud/apm/sleuth-thread-provider.png)

---

# Sleuth 拦截器、链路

## TraceFilter

Sleuth 通过 TraceFilter 获取 Span 信息。需要注意：如果需要对 Span 信息做自定义修改，需要实现自己的 Filter。
实现的 Filter 优先级需要比 TraceFilter 优先级低，否则无法拿到 TraceFilter 处理后的信息。

**Consumer**
```java
@Component
@Order(TraceWebServletAutoConfiguration.TRACING_FILTER_ORDER + 1) // 优先级低
public class SessionFilter extends GenericFilterBean {

    private Pattern pattern = Pattern.compile(SleuthWebProperties.DEFAULT_SKIP_PATTERN);

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain filterChain) throws IOException, ServletException {
        if (!(request instanceof HttpServletRequest) || !(response instanceof HttpServletResponse)) {
            throw new ServletException("只支持 Http 请求");
        }

        HttpServletRequest httpServletRequest = (HttpServletRequest) request;
        boolean matches = pattern.matcher(httpServletRequest.getRequestURI()).matches();

        if (!matches){
            // 向链路传入 SessionId，在 Provider 中获取
            ExtraFieldPropagation.set("SessionId", httpServletRequest.getSession().getId());
        }
        filterChain.doFilter(request, response);
    }
}
```

**Provider**
```java
@GetMapping(value = "/say")
    public String hello(String name){
        return "你好啊！" + name + ", 你的 session id 是：" + ExtraFieldPropagation.get("SessionId");
    }
```

## Baggage

Baggage 是存储在 Span 上下文中的一组 K/V 键值对，和 traceId、spanId 不同，baggage 不是必选项。
通过 Baggage 可以把一些信息像行李一样，挂在 sleuth 中，由 Sleuth 帮助沿着调用一直向下传递。
Baggage 相当于 Sleuth 暴露的一个功能接口，通过它，可以让数据跟着 Sleuth 一直往后接连传递，典型场景是登录信息的传递。

```yml
server:
  port: 8081
spring:
  sleuth:
    baggage-keys:
      - SessionId # 配置 Baggage 需要传递的参数名称，需要在传递端，接收端都配置。传递端不配置，不能向后传递；接收端不配置，获取不到数据。
```