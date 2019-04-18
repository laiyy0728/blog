---
title: Spring Cloud 微服务（17） ---Hystrix(五) <BR> Hystrix 优化
date: 2019-02-11 17:13:51
updated: 2019-02-11 17:13:51
categories:
    spring-cloud
tags:
    - Hystrix
    - SpringCloud
---


Hystrix Collapser 是 Hystrix 退出的针对多个请求，调用单个后端依赖的一种优化和节约网络开销的方法。在一般情况下，每个请求会开启一个线程，并开启一个对服务调用的网络连接，而 Collapser 可以将多个请求的线程合并起来，只需要一个线程和一个网络开销来调用服务，大大减少了并发和请求执行所需的线程数和网络连接，尤其是在一个时间段内非常多的请求情况下，能极大提高资源利用率。

<!-- more -->

# Hystrix Collapser

***源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-hystrix/spring-cloud-hsytrix-collapser***

## 实例

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
    name: spring-cloud-hystrix-collapser

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka
  instance:
    instance-id: ${spring.application.name}:${server.port}
    prefer-ip-address: true
```

### service、config、controller

```java
public interface ICollapserService {

    Future<Animal> collapsing(Integer id);

    Animal collapsingSyn(Integer id);

    Future<Animal> collapsingGlobal(Integer id);

}
```

```java
@Component
public class CollpaserServiceImpl implements ICollapserService {

    @Override
    @HystrixCollapser(batchMethod = "collapsingList", collapserProperties = {
            @HystrixProperty(name = HystrixPropertiesManager.TIMER_DELAY_IN_MILLISECONDS, value = "1000")
    })
    public Future<Animal> collapsing(Integer id) {
        return null;
    }

    @Override
    @HystrixCollapser(batchMethod = "collapsingList", collapserProperties = {
            @HystrixProperty(name = HystrixPropertiesManager.TIMER_DELAY_IN_MILLISECONDS, value = "1000")
    })
    public Animal collapsingSyn(Integer id) {
        return null;
    }

    @Override
    @HystrixCollapser(batchMethod = "collapsingListGlobal",
            scope = com.netflix.hystrix.HystrixCollapser.Scope.GLOBAL,
            collapserProperties = {
                    @HystrixProperty(name = HystrixPropertiesManager.TIMER_DELAY_IN_MILLISECONDS, value = "10000")
            })
    public Future<Animal> collapsingGlobal(Integer id) {
        return null;
    }

    @HystrixCommand
    public List<Animal> collapsingList(List<Integer> animalParam) {
        System.out.println("collapsingList 当前线程：" + Thread.currentThread().getName());
        System.out.println("当前请求参数个数：" + animalParam.size());
        List<Animal> animalList = Lists.newArrayList();
        animalParam.forEach(num -> {
            Animal animal = new Animal();
            animal.setName(" Cat - " + num);
            animal.setAge(num);
            animal.setSex("male");
            animalList.add(animal);
        });
        return animalList;
    }

    @HystrixCommand
    public List<Animal> collapsingListGlobal(List<Integer> animalParam) {
        System.out.println("collapsingList 当前线程：" + Thread.currentThread().getName());
        System.out.println("当前请求参数个数：" + animalParam.size());
        List<Animal> animalList = Lists.newArrayList();
        animalParam.forEach(num -> {
            Animal animal = new Animal();
            animal.setName(" Dog - " + num);
            animal.setAge(num);
            animal.setSex("female");
            animalList.add(animal);
        });
        return animalList;
    }

}
```

```java
@Configuration
public class CollapserConfiguration {


    @Bean
    @ConditionalOnClass(Controller.class)
    public HystrixCollapserInterceptor hystrixCollapserInterceptor(){
        return new HystrixCollapserInterceptor();
    }

    @Configuration
    @ConditionalOnClass(Controller.class)
    public class WebMvcConfig extends WebMvcConfigurationSupport{
        private final HystrixCollapserInterceptor interceptor;

        @Autowired
        public WebMvcConfig(HystrixCollapserInterceptor interceptor) {
            this.interceptor = interceptor;
        }

        @Override
        protected void addInterceptors(InterceptorRegistry registry) {
            registry.addInterceptor(interceptor);
        }
    }

}
```

```java
@RestController
public class CollapserController {

    private final ICollapserService collapserService;

    @Autowired
    public CollapserController(ICollapserService collapserService) {
        this.collapserService = collapserService;
    }

    @GetMapping("/get-animal")
    public String getAnimal() throws Exception{
        Future<Animal> animal = collapserService.collapsing(1);
        Future<Animal> animal2 = collapserService.collapsing(2);
        System.out.println(animal.get().getName());
        System.out.println(animal2.get().getName());
        return "success";
    }

    @GetMapping(value = "/get-animal-syn")
    public String getAnimalSyn() throws Exception{
        Animal animal1 = collapserService.collapsingSyn(1);
        Animal animal2 = collapserService.collapsingSyn(2);
        System.out.println(animal1.getName());
        System.out.println(animal2.getName());
        return "success";
    }

    @GetMapping(value = "/get-animal-global")
    public String getAnimalGlobal() throws Exception {
        Future<Animal> animal1 = collapserService.collapsingGlobal(1);
        Future<Animal> animal2 = collapserService.collapsingGlobal(2);
        System.out.println(animal1.get().getName());
        System.out.println(animal2.get().getName());
        return "success";
    }

}
```


### 验证

访问： http://localhost:8989/get-animal 、http://localhost:8989/get-animal-syn 、 http://localhost:8989/get-animal-global ，查看控制台：

http://localhost:8989/get-animal：
```
collapsingList 当前线程：hystrix-CollpaserServiceImpl-1
当前请求参数个数：2
 Cat - 1
 Cat - 2
```
http://localhost:8989/get-animal-syn：
```
collapsingList 当前线程：hystrix-CollpaserServiceImpl-2
当前请求参数个数：1
collapsingList 当前线程：hystrix-CollpaserServiceImpl-3
当前请求参数个数：1
 Cat - 1
 Cat - 2
```
http://localhost:8989/get-animal-global：
```
collapsingListGlobal 当前线程：hystrix-CollpaserServiceImpl-4
当前请求参数个数：2
 Dog - 1
 Dog - 2
```

可以看到，get-animal、gei-animal-global 合并了请求，get-animal-syn 没有合并请求

## 注意点

- 要合并的请求，必须返回 Future，通过在实现类上增加 `@HystrixCollapser` 注解，之后调用该方法来实现请求合并。（需要特别注意：方法返回值不是 Future 无法合并请求）
- `@HystrixCollapser` 表示合并请求，调用该注解标注的方法时，实际上调用的是 `batchMethod` 方法，且利用 `HystrixProperty` 指定 `timerDelayInMilliseconds`，表示合并多少 ms 之内的请求。默认是 10ms
- 如果不在 `@HystrixCollapser` 中添加 scope=GLOBAL ，则只会合并服务的多次请求。当 `scope=GLOBAL` 时，会合并规定时间内的所有服务的多次请求。

scope 有两个值， `REQUEST\GLOBAL`。 REQUEST 表示合并单个服务的调用，GLOBAL 表示合并所有服务的调用。默认为 REQUEST。

## 总结

Hystrix Collapser 主要用于请求合并的场景。当在某个时间内有大量或并发的相同请求时，适用于请求合并；如果在某个时间内只有很少的请求，且延迟也不高，使用请求反而会增加复杂度和延迟，Collapser 本身也需要时间进行处理。

---

# Hystrix 线程传递、并发

***源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-hystrix/spring-cloud-hystrix-thread***

Hystrix 的两种隔离策略：线程隔离、信号量隔离。

如果是信号量隔离，则 Hystrix 在请求时会尝试获取一个信号量，如果成功拿到，则继续请求，该请求在一个线程内完成。如果是线程隔离，Hystrix 会把请求放入线程池执行，这是可能产生线程变化，导致`线程1`的上下文在`线程2`中无法获取到。

## 不适应 Hystrix 调用

pom、yml 等不再赘述

### service、controller

```java
// 接口
public interface IThreadService {
    String getUser(int id);
}


// 具体调用实现
@Component
public class ThreadServiceImpl implements IThreadService {

    private static final Logger LOGGER = LoggerFactory.getLogger(ThreadServiceImpl.class);

    private final RestTemplate restTemplate;

    @Autowired
    public ThreadServiceImpl(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    @Override
    public String getUser(int id) {
        LOGGER.info("================================== Service ==================================");
        LOGGER.info(" ThreadService, Current thread id : {}", Thread.currentThread().getId());
        LOGGER.info(" ThreadService, HystrixThreadLocal: {}", HystrixThreadLocal.threadLocal.get());
        LOGGER.info(" ThreadService, RequestContextHolder: {}", RequestContextHolder.currentRequestAttributes().getAttribute("userId", RequestAttributes.SCOPE_REQUEST));
        // 这里调用的是 spring-cloud-hystrix-cache 的 provider
        return restTemplate.getForObject("http://spring-cloud-hystrix-cache-provider-user/get-user/{1}", String.class, id);
    }
}


// thread local
public class HystrixThreadLocal {
    public static ThreadLocal<String> threadLocal = new InheritableThreadLocal<>();
}


// controller
@RestController
public class ThreadController {

    private static final Logger LOGGER = LoggerFactory.getLogger(ThreadController.class);

    private final IThreadService threadService;

    @Autowired
    public ThreadController(IThreadService threadService) {
        this.threadService = threadService;
    }

    @GetMapping(value = "/get-user/{id}")
    public String getUser(@PathVariable int id) {
        // 放入上下文
        HystrixThreadLocal.threadLocal.set("userId:" + id);
        // 利用 RequestContextHolder
        RequestContextHolder.currentRequestAttributes().setAttribute("userId", "userId:" + id, RequestAttributes.SCOPE_REQUEST);
        LOGGER.info("================================== Controller ==================================");
        LOGGER.info(" ThreadService, Current thread id : {}", Thread.currentThread().getId());
        LOGGER.info(" ThreadService, HystrixThreadLocal: {}", HystrixThreadLocal.threadLocal.get());
        LOGGER.info(" ThreadService, RequestContextHolder: {}", RequestContextHolder.currentRequestAttributes().getAttribute("userId", RequestAttributes.SCOPE_REQUEST));
        return threadService.getUser(id);
    }

}
```

### 验证

访问 http://localhost:8989/get-user/1 ，查看控制台输出
```
================================== Controller ==================================
ThreadService, Current thread id : 37
ThreadService, HystrixThreadLocal: userId:1
ThreadService, RequestContextHolder: userId:1
================================= Service ==================================
ThreadService, Current thread id : 37
ThreadService, HystrixThreadLocal: userId:1
ThreadService, RequestContextHolder: userId:1
```

可以看到， thread id 是一样的，userid 也是一样的，这说明了此时是使用一个线程进行调用。


## Hystrix 分线程调用

在 Service 的实现类上增加 @HystrixCommand 注解，重启再次访问，查看可控制台输出如下：
```
================================== Controller ==================================
ThreadService, Current thread id : 36
ThreadService, HystrixThreadLocal: userId:1
ThreadService, RequestContextHolder: userId:1
================================= Service ==================================
ThreadService, Current thread id : 60
ThreadService, HystrixThreadLocal: userId:1
```

并在 `ThreadService, RequestContextHolder` 的位置抛出如下异常：
```
java.lang.IllegalStateException: No thread-bound request found: Are you referring to request attributes outside of an actual web request, or processing a request outside of the originally receiving thread? If you are actually operating within a web request and still receive this message, your code is probably running outside of DispatcherServlet: In this case, use RequestContextListener or RequestContextFilter to expose the current request.
	at org.springframework.web.context.request.RequestContextHolder.currentRequestAttributes(RequestContextHolder.java:131) ~[spring-web-5.1.2.RELEASE.jar:5.1.2.RELEASE]
	at com.laiyy.gitee.hystrix.thread.springcloudhystrixthread.service.ThreadServiceImpl.getUser(ThreadServiceImpl.java:36) ~[classes/:na]
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[na:1.8.0_171]
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62) ~[na:1.8.0_171]
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:1.8.0_171]
	at java.lang.reflect.Method.invoke(Method.java:498) ~[na:1.8.0_171]
	at com.netflix.hystrix.contrib.javanica.command.MethodExecutionAction.execute(MethodExecutionAction.java:116) ~[hystrix-javanica-1.5.12.jar:1.5.12]
	at com.netflix.hystrix.contrib.javanica.command.MethodExecutionAction.executeWithArgs(MethodExecutionAction.java:93) ~[hystrix-javanica-1.5.12.jar:1.5.12]
    ...
```

可以看到，在 controller 中，线程id是 36，在 service 中，线程id是 60，线程id不一致，可以证明线程隔离已经生效了。此时 controller 调用 @HystrixCommand 标注的方法时，是分线程处理的；RequestContextHolder 中报错：没有线程变量绑定。

## 解决办法

解决办法有两种：修改 Hystrix 隔离策略，使用信号量即可；使用 HystrixConcurrencyStrategy(官方推荐)

使用 HystrixConcurrencyStrategy 实现 wrapCallable 方法，对于依赖的 ThreadLocal 状态以实现应用程序功能的系统至关重要。

### 官方文档

wrapCallable 方法源码：
```java
/**
 * Provides an opportunity to wrap/decorate a {@code Callable<T>} before execution.
 * <p>
 * This can be used to inject additional behavior such as copying of thread state (such as {@link ThreadLocal}).
 * <p>
 * <b>Default Implementation</b>
 * <p>
 * Pass-thru that does no wrapping.
 * 
 * @param callable
 *            {@code Callable<T>} to be executed via a {@link ThreadPoolExecutor}
 * @return {@code Callable<T>} either as a pass-thru or wrapping the one given
 */
public <T> Callable<T> wrapCallable(Callable<T> callable) {
    return callable;
}
```

通过重写这个方法，实现想要封装的线程参数方法。
可以看到，返回值是一个 Callable，所以需要首先实现一个 Callable 类，在该类中，在执行请求之前，包装 HystrixThreadCallable 对象，传递 `RequestContextHolder`、`HystrixThreadLocal` 类，将需要的对象信息设置进去，这样在下一个线程中就可以拿到了。

### 具体实现

```java
// Callable
public class HystrixThreadCallable<S> implements Callable<S> {

    private final RequestAttributes requestAttributes;
    private final Callable<S> callable;
    private String params;

    public HystrixThreadCallable(RequestAttributes requestAttributes, Callable<S> callable, String params) {
        this.requestAttributes = requestAttributes;
        this.callable = callable;
        this.params = params;
    }

    @Override
    public S call() throws Exception {
        try {
            // 在执行请求之前，包装请求参数
            RequestContextHolder.setRequestAttributes(requestAttributes);
            HystrixThreadLocal.threadLocal.set(params);
            // 执行具体请求
            return callable.call();
        } finally {
            RequestContextHolder.resetRequestAttributes();
            HystrixThreadLocal.threadLocal.remove();
        }
    }
}


// HystrixConcurrencyStrategy
public class SpringCloudHystrixConcurrencyStrategy extends HystrixConcurrencyStrategy {

    private HystrixConcurrencyStrategy hystrixConcurrencyStrategy;

    @Override
    public <T> Callable<T> wrapCallable(Callable<T> callable) {
        // 传入需要传递到分线程的参数
        return new HystrixThreadCallable<>(RequestContextHolder.currentRequestAttributes(), callable, HystrixThreadLocal.threadLocal.get());
    }

    public SpringCloudHystrixConcurrencyStrategy() {
        init();
    }

    private void init() {
        try {
            this.hystrixConcurrencyStrategy = HystrixPlugins.getInstance().getConcurrencyStrategy();

            if (this.hystrixConcurrencyStrategy instanceof SpringCloudHystrixConcurrencyStrategy){
                return;
            }

            HystrixCommandExecutionHook commandExecutionHook = HystrixPlugins.getInstance().getCommandExecutionHook();
            HystrixEventNotifier eventNotifier = HystrixPlugins.getInstance().getEventNotifier();
            HystrixMetricsPublisher metricsPublisher = HystrixPlugins.getInstance().getMetricsPublisher();
            HystrixPropertiesStrategy propertiesStrategy = HystrixPlugins.getInstance().getPropertiesStrategy();

            HystrixPlugins.reset();

            HystrixPlugins.getInstance().registerConcurrencyStrategy(this);
            HystrixPlugins.getInstance().registerCommandExecutionHook(commandExecutionHook);
            HystrixPlugins.getInstance().registerEventNotifier(eventNotifier);
            HystrixPlugins.getInstance().registerMetricsPublisher(metricsPublisher);
            HystrixPlugins.getInstance().registerPropertiesStrategy(propertiesStrategy);
        }catch (Exception e){
            throw e;
        }
    }

    @Override
    public ThreadPoolExecutor getThreadPool(HystrixThreadPoolKey threadPoolKey, HystrixProperty<Integer> corePoolSize, HystrixProperty<Integer> maximumPoolSize, HystrixProperty<Integer> keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue) {
        return this.hystrixConcurrencyStrategy.getThreadPool(threadPoolKey, corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue);
    }

    @Override
    public ThreadPoolExecutor getThreadPool(HystrixThreadPoolKey threadPoolKey, HystrixThreadPoolProperties threadPoolProperties) {
        return this.hystrixConcurrencyStrategy.getThreadPool(threadPoolKey, threadPoolProperties);
    }

    @Override
    public BlockingQueue<Runnable> getBlockingQueue(int maxQueueSize) {
        return this.hystrixConcurrencyStrategy.getBlockingQueue(maxQueueSize);
    }

    @Override
    public <T> HystrixRequestVariable<T> getRequestVariable(HystrixRequestVariableLifecycle<T> rv) {
        return this.hystrixConcurrencyStrategy.getRequestVariable(rv);
    }
}
```

## 验证

重启后访问： http://localhost:8989/get-user/1 ，查看控制台输出
```
================================= Controller ==================================
ThreadService, Current thread id : 39
ThreadService, HystrixThreadLocal: userId:1
ThreadService, RequestContextHolder: userId:1
================================= Service ==================================
ThreadService, Current thread id : 73
ThreadService, HystrixThreadLocal: userId:1
ThreadService, RequestContextHolder: userId:1
```

可以看到，线程id不一样，但是 RequestContextHolder、HystrixThreadLocal 的值都可以在分线程中拿到了。

## 注意点

在 `SpringCloudHystrixConcurrencyStrategy#init()` 中，`HystrixPlugins.getInstance().registerConcurrencyStrategy()` 方法，只能被调用一次，否则会出现错误。
源码解释：
```java
/**
 * Register a {@link HystrixConcurrencyStrategy} implementation as a global override of any injected or default implementations.
 * 
 * @param impl
 *            {@link HystrixConcurrencyStrategy} implementation
 * @throws IllegalStateException
 * // 如果多次调用或在初始化默认值之后（如果在尝试注册之前发生了使用） 会抛出异常
 *             if called more than once or after the default was initialized (if usage occurs before trying to register)
 */
public void registerConcurrencyStrategy(HystrixConcurrencyStrategy impl) {
    if (!concurrencyStrategy.compareAndSet(null, impl)) {
        throw new IllegalStateException("Another strategy was already registered.");
    }
}
```

---

# Hystrix 命令注解

## HystrixCommand

作用：封装执行的代码，具有故障延迟容错、断路器、统计等功能，是阻塞命令，可以与 Observable 公用

com.netflix.hystrix.HystrixCommand 类
```java
/**
 * Used to wrap code that will execute potentially risky functionality (typically meaning a service call over the network)
 * with fault and latency tolerance, statistics and performance metrics capture, circuit breaker and bulkhead functionality.
 * This command is essentially a blocking command but provides an Observable facade if used with observe()
 * 
 * @param <R>
 *            the return type
 * 
 * @ThreadSafe
 */
public abstract class HystrixCommand<R> extends AbstractCommand<R> implements HystrixExecutable<R>, HystrixInvokableInfo<R>, HystrixObservable<R> {
}
```

用于包装将执行潜在风险功能的代码（通常意味着通过网络进行服务调用）具有故障和延迟限，统计和性能指标捕获，断路器和隔板功能。
该命令本质上是一个阻塞命令，但如果与observe（）一起使用，则提供一个Observable外观

## HystrixObservableCommand

```java
/**
 * Used to wrap code that will execute potentially risky functionality (typically meaning a service call over the network)
 * with fault and latency tolerance, statistics and performance metrics capture, circuit breaker and bulkhead functionality.
 * This command should be used for a purely non-blocking call pattern. The caller of this command will be subscribed to the Observable<R> returned by the run() method.
 * 
 * @param <R>
 *            the return type
 * 
 * @ThreadSafe
 */
public abstract class HystrixObservableCommand<R> extends AbstractCommand<R> implements HystrixObservable<R>, HystrixInvokableInfo<R> {
}
```
用于包装将执行潜在风险功能的代码（通常意味着通过网络进行服务调用）具有故障和延迟限，统计和性能指标捕获，断路器和隔板功能。
此命令应该用于纯粹的非阻塞调用模式。 此命令的调用者将订阅run（）方法返回的Observable <R>。


## 区别

- HystrixCommand 默认是阻塞的可以提供同步、异步两种方式；HystrixObservableCommand 是非阻塞的，只能提供异步的方式
- HystrixCommand 的方法是 run；HystrixObservableCommand 的方法是 construct
- HystrixCommand 一个实例一次只能发一条数据，HystrixObservableCommand 可以发送多条数据


HystrixCommand 注解：
![HystrixCommand 注解](/images/spring-cloud/hystrix/hystrix-command.png)

- commandKey：全局唯一标识符，如果不设置，默认是方法名
- defaultFallback：默认 fallback 方法，不能有入参，返回值和方法保持一致，比 fallbackMethod 的优先级低
- fallbackMethod：指定处理回退逻辑的方法，必须和 HystrixCommand 标注的方法在通一个类里，参数、返回值要保持一致
- ignoreExceptions：定义不希望哪些异常被 fallback，而是直接抛出
- commandProperties：配置命名属性，如隔离策略
- threadPoolProperties：配置线程池相关属性
- groupKey：全局唯一分组名称，内部会根据这个值展示统计数、仪表盘等，默认使的线程划分是根据组名称进行的，一般会在创建 HystrixCommand 时指定组来实现默认的线程池划分
- threadPoolKey：对服务的线程池信息进行设置，用于 HystrixThreadPool 监控、metrics、缓存等。