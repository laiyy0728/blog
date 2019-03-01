---
title: Spring Cloud 微服务（26） --- Zuul(七) <BR> Zuul 优化、Zuul 工作原理
date: 2019-02-25 16:45:46
updated: 2019-02-25 16:45:46
categories:
    Java
tags: [SpringCloud, Zuul]
---


Zuul 作为一个网关中间件，需要应付各种复杂场景，整合的组件非常繁杂。在受益于其丰富的功能时，也需要面对很多问题。如：与上层负载均衡器(Nginx等)、性能、调优等。

<!-- more -->

# Zuul 应用优化

Zuul 是建立在 Servlet 上的同步阻塞架构，所有在处理逻辑上面是和线程密不可分，每一次请求都需要在线程池获取一个线程来维护 I/O 操作，路由转发的时候又需要从 http 客户端获取线程来维持连接，这样会导致一个组件占用两个线程资源的情况。所以在 Zuul 的使用中，对这部分的优化很有必要。

Zuul 的优化分为以下几个类型：
- 容器优化：内置容器 tomcat 与 undertow 的比较与参数设置
- 组件优化：内部集成的组件优化，如 Hystrix 线程隔离、Ribbon、HttpClient、OkHttp 选择等
- JVM 参数优化：适用于网关应用的 JVM 参数建议
- 内部优化：内部原生参数，内部源码，重写等


## 容器优化

把 tomcat 替换为 undertow。`undertow` 翻译为“暗流”，是一个轻量级、高性能容器。`undertow` 提供阻塞或基于 XNIO 的非阻塞机制，包大小不足 1M，内嵌模式运行时的堆内存占用只有 4M。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>
</dependency>
```

```yml
server:
  undertow:
    io-threads: 10
    worker-threads: 10
    direct-buffers: true
    buffer-size: 1024 # 字节数
```

| 配置项 | 默认值 | 说明 |
| :-: | :-: | :-: |
| server.undertow.io-threads | `Math.max(Runtime.getRuntime().availableProcessors(), 2)` | 设置 IO 线程数，它主要执行非阻塞的任务，它们负责多个连接，默认设置每个 CPU 核心有一个线程。不要设置太大，否则启动项目会报错：打开文件数过多 |
| server.undertow.worker-threads | io-threads * 8 | 阻塞任务线程数，当执行类型 Servlet 请求阻塞 IO 操作，undertow 会从这个线程池中取得线程。值设置取决于系统线程执行任务的阻塞系统，默认是 IO 线程数 * 8 |
| server.undertow.direct-buffers | 取决于 JVM 最大可用内存大小`Runtime.getRuntime().maxMemory()`，小于 64MB 默认为 false，其余默认为 true | 是否分配直接内存（NIO 直接分配的堆外内存 |
| server.undertow.buffer-size | 最大可用内存 <64MB：512 字节；64MB< 最大可用内存 <128MB：1024 字节；128MB < 最大可用内存：1024*16 - 20 字节 | 每块 buffer 的空间大小，空间越小利用越充分，设置太大会影响其他应用 |
| server.undertow.buffers-per-region | 最大可用内存 <128MB：10；128MB < 最大可用内存：20 | 每个区域分配的 buffer 数量，pool 大小是 buffer-size * buffer-per-region |

## 组件优化

### Hystrix

在 Zuul 中默认集成了 Hystrix 熔断器，使得网关应用具有弹性、容错的能力。但是如果使用默认配置，可能会遇到问题。如：第一次请求失败。这是因为第一次请求的时候，zuul 内部需要初始化很多信息，十分耗时。而 hystrix 默认超时时间是一秒，可能会不够。

解决方式：
- 加大超时时间
```yml
hystrix:
 command:
   default: 
     execution:
       isolation:
         thread:
           timeoutInMilliseconds: 5000 
```

- 禁用 hystrix 超时
```yml
hystrix:
 command:
   default: 
     execution:
       timeout:
         enabled: false    
```

Zuul 中关于 Hystrix 的配置还有一个很重要的点：`Hystrix 线程隔离策略`。

| | 线程池模式(THREAD) | 信号量模式(SEMAPHORE) |
| :-: | :-: | :-: |
| 官方推荐 | 是 | 否 |
| 线程 | 与请求线程分离 | 与请求线程公用 |
| 开销 | 上下文切换频繁，较大 | 较小 |
| 异步 | 支持 | 不支持 |
| 应对并发量 | 大 | 小 |
| 适用场景 | 外网交互 | 内网交互 |



如果应用需要与外网交互，由于网络开销比较大、请求比较耗时，选用线程隔离，可以保证有剩余容器（tomcat 等）线程可用，不会由于外部原因使线程一直在阻塞或等待状态，可以快速返回失败
如果应用不需要与外网交互，并且体量较大，使用信号量隔离，这类应用响应通常非常快，不会占用容器线程太长时间，使用信号量线程上下文就会成为一个瓶颈，可以减少线程切换的开销，提高应用运转的效率，也可以气到对请求进行全局限流的作用。


### Ribbon

```yml
ribbon:
  ConnectTimeout: 3000
  ReadTimeout: 60000
  MaxAutoRetries: 1 # 对第一次请求的发我的重试次数
  MaxAutoRetriesNextServer: 1 # 要重试的下一个服务的最大数量（不包括第一个服务）
  OkToRetryOnAllOperations: true
```

`ConnectTimeout`、`ReadTimeout` 是当前 HTTP 客户端使用 HttpClient 的时候生效的，这个超时时间最终会被设置到 HttpClient 中。在设置的时候要结合 Hystrix 超时时间综合考虑。设置太小会导致请求失败，设置太大会导致 Hystrix 熔断控制变差。


## JVM 参数优化

根据实际情况，调整 JVM 参数

## 内有优化

在官方文档中，zuul 部分将 `zuul.max.host.coonnections` 属性拆分成了 `zuul.host.maxTotalConnections`、`zuul.host.maxPerRouteConnections`，默认值分别为 200、20。
需要注意：这个配置只在使用 HttpClient 时有效，使用 OkHttp 无效。

zuul 中还有一个超时时间，使用 serviceId 映射与 url 映射的设置是不一样的，如果使用 serviceId 映射，`ribbon.ReadTimeout` 与 `ribbon.SocketTimeout` 生效；如果使用 url 映射，`zuul.host.connect-timeout-millis` 与 `zuul.host.socket-timeout-millis` 生效

---

# Zuul 原理、核心

zuul 官方提供了一张架构图，很好的描述了 Zuul 工作原理

![Zuul 架构图](/images/spring-cloud/zuul/zuul-framework.png)

Zuul Servlet 通过 RequestContext 通关着由许多 Filter 组成的核心组件，所有操作都与 Filter 息息相关。请求、ZuulServlet、Filter 共同构建器 Zuul 的运行时声明周期

![zuul life cycle](/images/spring-cloud/zuul/zuul-life-cycle-2.png)

Zuul 的请求来自于 DispatcherServlet，然后交给 ZuulHandlerMapping 处理初始化得来的路由定位器`RouteLocator`，为后续的请求分发做好准备，同时整合了基于事件从服务中心拉取服务列表的机制；
进入 ZuulController，主要职责是初始化 ZuulServlet 以及集成 ServletWrappingController，通过重写 handleRequest 方法来将 ZuulServlet 引入声明周期，之后所有的请求都会经过 ZuulServlet；
当请求进入 ZuulServlet 之后，第一次调用会初始化 ZuulRunner，非第一次调用就按照 Filter 链的 order 顺序执行；
ZuulRunner 中将请求和响应初始化为 RequestContext，包装成 FilterProcessor 转换为为调用 preRoute、route、postRoute、error 方法；
最后再 Filter 链中经过种种变换，得到预期结果。

## EnableZuulProxy、EnableZuulServer

对比 `EnableZuulProxy`、`EnableZuulServer`
```java
@EnableCircuitBreaker
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Import(ZuulProxyMarkerConfiguration.class)
public @interface EnableZuulProxy {
}
```

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import({ZuulServerMarkerConfiguration.class})
public @interface EnableZuulServer {
}
```

这两个注解的区别在于 `@Import` 中的配置类不一样。查看两个配置类的源码：
```java
/**
 * Responsible for adding in a marker bean to trigger activation of 
 * {@link ZuulProxyAutoConfiguration}
 *
 * @author Biju Kunjummen
 */

@Configuration
public class ZuulProxyMarkerConfiguration {
	@Bean
	public Marker zuulProxyMarkerBean() {
		return new Marker();
	}

	class Marker {
	}
}
```

```java
/**
 * Responsible for adding in a marker bean to trigger activation of 
 * {@link ZuulServerAutoConfiguration}
 *
 * @author Biju Kunjummen
 */

@Configuration
public class ZuulServerMarkerConfiguration {
	@Bean
	public Marker zuulServerMarkerBean() {
		return new Marker();
	}

	class Marker {
	}
}
```


可以看到，这两个配置类的源码一致，区别在于类的注释上 `@link` 指向的自动装配类不一样，`ZuulProxyMarkerConfiguration` 对应的是 `ZuulProxyAutoConfiguration`；`ZuulServerMarkerConfiguration` 对应的是 `ZuulServerAutoConfiguration`

查看 `ZuulProxyAutoConfiguration` 和 `ZuulServerAutoConfiguration` 的类注解

```java
@Configuration
@Import({ RibbonCommandFactoryConfiguration.RestClientRibbonConfiguration.class,
		RibbonCommandFactoryConfiguration.OkHttpRibbonConfiguration.class,
		RibbonCommandFactoryConfiguration.HttpClientRibbonConfiguration.class,
		HttpClientConfiguration.class })
@ConditionalOnBean(ZuulProxyMarkerConfiguration.Marker.class)
public class ZuulProxyAutoConfiguration extends ZuulServerAutoConfiguratio
}
```

```java
@Configuration
@EnableConfigurationProperties({ ZuulProperties.class })
@ConditionalOnClass({ZuulServlet.class, ZuulServletFilter.class})
@ConditionalOnBean(ZuulServerMarkerConfiguration.Marker.class)
public class ZuulServerAutoConfiguration {
}
```

可以发现，这两个类是通过 `ZuulServerMarkerConfiguration`、`ZuulProxyMarkerConfiguration` 中 Marker 类是否存在，当做是否进行自动装配的开关。
对比两个 `AutoCOnfiguration` 的具体源码实现，经过对比，可以分析出：
`ZuulServerAutoConfiguration` 的功能是：
- 初始化配置加载器
- 初始化路由定位器
- 初始化路由映射器
- 初始化配置刷新监听器
- 初始化 ZuulServlet 加载器
- 初始化 ZuulController
- 初始化 Filter 执行解析器
- 初始化部分 Filter
- 初始化 Metrix 监控

`ZuulProxyAutoConfiguration` 的功能是：
- 初始化服务注册、发现监听器
- 初始化服务列表监听器
- 初始化 zuul 自定义的 endpoint
- 初始化一些 `ZuulServerAutoConfiguration` 中没有的 filter'
- 引入 http 客户端的两种方式：HttpClient、OkHttp

## Filter 链

### filter 装载 

zuul 中的 Filter 必须经过初始化装载，才能在请求中发挥作用，其过程如下
![zuul-filter-life-cycle](/images/spring-cloud/zuul/zuul-filter-life-cycle.png)

#### zuul filter 连初始化过程
```java
public class ZuulServerAutoConfiguration {

  // 省略其他代码

  @Configuration
  protected static class ZuulFilterConfiguration {
    @Autowired
    private Map<String, ZuulFilter> filters;
    @Bean
    public ZuulFilterInitializer zuulFilterInitializer(
      CounterFactory counterFactory, TracerFactory tracerFactory) {
      FilterLoader filterLoader = FilterLoader.getInstance();
      FilterRegistry filterRegistry = FilterRegistry.instance();
      return new ZuulFilterInitializer(this.filters, counterFactory, tracerFactory, filterLoader, filterRegistry);
    }
  }
}
```

```java
public class ZuulFilterInitializer {

  // 省略其他代码

  // @PostConstruct：表明在 Bean 初始化之前，就把 Filter 的信息保存到 FilterRegistry
	@PostConstruct
	public void contextInitialized() {
		log.info("Starting filter initializer");

		TracerFactory.initialize(tracerFactory);
		CounterFactory.initialize(counterFactory);

		for (Map.Entry<String, ZuulFilter> entry : this.filters.entrySet()) {
			filterRegistry.put(entry.getKey(), entry.getValue());
		}
	}

  // @PreDestroy：表明在 bean 销毁之前清空 filterRegistry 与 FilterLoader。filterloader 可以通过 filter 名、filter class、filter 类型来查询得到相应的 filter
	@PreDestroy
	public void contextDestroyed() {
		log.info("Stopping filter initializer");
		for (Map.Entry<String, ZuulFilter> entry : this.filters.entrySet()) {
			filterRegistry.remove(entry.getKey());
		}
		clearLoaderCache();

		TracerFactory.initialize(null);
		CounterFactory.initialize(null);
	}

  private void clearLoaderCache() {
		Field field = ReflectionUtils.findField(FilterLoader.class, "hashFiltersByType");
		ReflectionUtils.makeAccessible(field);
		@SuppressWarnings("rawtypes")
		Map cache = (Map) ReflectionUtils.getField(field, filterLoader);
		cache.clear();
	}
}
```

#### zuul filter 请求调用过
```java
public class ZuulServletFilter implements Filter {

    private ZuulRunner zuulRunner;

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {

        String bufferReqsStr = filterConfig.getInitParameter("buffer-requests");
        boolean bufferReqs = bufferReqsStr != null && bufferReqsStr.equals("true") ? true : false;

        zuulRunner = new ZuulRunner(bufferReqs);
    }

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        try {
            init((HttpServletRequest) servletRequest, (HttpServletResponse) servletResponse);
            try {
                preRouting();
            } catch (ZuulException e) {
                error(e);
                postRouting();
                return;
            }
            
            // Only forward onto to the chain if a zuul response is not being sent
            if (!RequestContext.getCurrentContext().sendZuulResponse()) {
                filterChain.doFilter(servletRequest, servletResponse);
                return;
            }
            
            try {
                routing();
            } catch (ZuulException e) {
                error(e);
                postRouting();
                return;
            }
            try {
                postRouting();
            } catch (ZuulException e) {
                error(e);
                return;
            }
        } catch (Throwable e) {
            error(new ZuulException(e, 500, "UNCAUGHT_EXCEPTION_FROM_FILTER_" + e.getClass().getName()));
        } finally {
            RequestContext.getCurrentContext().unset();
        }
    }

    void postRouting() throws ZuulException {
        zuulRunner.postRoute();
    }
    
    // 省略其他代码
}
```

```java
public class ZuulRunn{

  // 省略其他代码

  public void postRoute() throws ZuulException {
        FilterProcessor.getInstance().postRoute();
  }

  // 省略其他代码
}
```

```java
public class FilterProcessor {
    public void postRoute() throws ZuulException {
        try {
            runFilters("post");
        } catch (ZuulException e) {
            throw e;
        } catch (Throwable e) {
            throw new ZuulException(e, 500, "UNCAUGHT_EXCEPTION_IN_POST_FILTER_" + e.getClass().getName());
        }
    }

    public Object runFilters(String sType) throws Throwable {
        if (RequestContext.getCurrentContext().debugRouting()) {
            Debug.addRoutingDebug("Invoking {" + sType + "} type filters");
        }
        boolean bResult = false;
        List<ZuulFilter> list = FilterLoader.getInstance().getFiltersByType(sType);
        if (list != null) {
            for (int i = 0; i < list.size(); i++) {
                ZuulFilter zuulFilter = list.get(i);
                Object result = processZuulFilter(zuulFilter);
                if (result != null && result instanceof Boolean) {
                    bResult |= ((Boolean) result);
                }
            }
        }
        return bResult;
    }

    // 省略其他代码
}
```

## 核心路由的实现

Zuul 的路由有一个顶级接口 `RouteLocator`。所有关于路由的功能都是由此而来，其中定义了三个方法：获取忽略的 path 集合、获取路由列表、根据 path 获取路由信息。
![Route 类图](/images/spring-cloud/zuul/route-locator.png)

`SimpleRouteLocator` 是一个基本实现，主要功能是对 ZuulServer 的配置文件中路由规则的维护，实现了 Ordered 接口，可以对定位器优先级进行设置。
Spring 是一个大量使用策略模式的框架，在策略模式下，接口的实现类有一个优先级问题，Spring 通过 Ordered 接口实现优先级。

`ZuulProperties$ZuulRoute` 类就是维护路由规则的类，具体属性如下：
```java
public static class ZuulRoute {

		private String id;

		private String path;

		private String serviceId;

		private String url;

		private boolean stripPrefix = true;
	
		private Boolean retryable;

		private Set<String> sensitiveHeaders = new LinkedHashSet<>();

		private boolean customSensitiveHeaders = false;
}
```

`RefreshableRouteLocator` 扩展了 RouteLocator 接口，在 ZuulHandlerMapping 中才实质性生效：凡是实现了 RefreshableRouteLocator，都会被时间监听器所刷新：
```java
public void setDirty(boolean dirty) {
		this.dirty = dirty;
		if (this.routeLocator instanceof RefreshableRouteLocator) {
			((RefreshableRouteLocator) this.routeLocator).refresh();
		}
	}
```

`DiscoveryClientRouteLocator` 实现了 RefreshableRouteLocator，扩展了 SimpleRouteLocator。其作用是整合配置文件与注册中心的路由信息。

`CompositeRouteLocator`，在 `ZuulServerAutoConfiguration` 中配置加载时，有一个很重要的注解：`@Primary`，表示所有的 `RouteLocator` 中，优先加载它，也就是说，所有的定位器都要在这里装配，可以看做其他路由定位器的处理器。
zuul 通过它来将请求域路由规则进行关联，这个操作在 `ZuulHandlerMapping` 中：
```java
public class ZuulHandlerMapping extends AbstractUrlHandlerMapping {

  // 省略其他代码

	private final RouteLocator routeLocator;

	private final ZuulController zuul;

  public ZuulHandlerMapping(RouteLocator routeLocator, ZuulController zuul) {
		this.routeLocator = routeLocator;
		this.zuul = zuul;
		setOrder(-200);
	}

  @Override
	protected Object lookupHandler(String urlPath, HttpServletRequest request) throws Exception {
		if (this.errorController != null && urlPath.equals(this.errorController.getErrorPath())) {
			return null;
		}
		if (isIgnoredPath(urlPath, this.routeLocator.getIgnoredPaths())) return null;
		RequestContext ctx = RequestContext.getCurrentContext();
		if (ctx.containsKey("forward.to")) {
			return null;
		}
		if (this.dirty) {
			synchronized (this) {
				if (this.dirty) {
					registerHandlers();
					this.dirty = false;
				}
			}
		}
		return super.lookupHandler(urlPath, request);
	}

  private void registerHandlers() {
		Collection<Route> routes = this.routeLocator.getRoutes();
		if (routes.isEmpty()) {
			this.logger.warn("No routes found from RouteLocator");
		}
		else {
			for (Route route : routes) {
				registerHandler(route.getFullPath(), this.zuul);
			}
		}
	}

}
```

ZuulHandlerMapping 将映射规则交给 `ZuulController` 处理，而 ZuulController 又到 ZuulServlet 中处理，最后到达异域或源服务发送 http 请求的 route 类型的 filter 中，默认有三种发送 http 请求的 filter
- RibbonRoutingFilter：优先级 10，使用 Ribbon、Hystrix、嵌入式 HTTP 客户端发送请求
- SimpleHostRoutingFilter：优先级 100，室友 Apache HttpClient 发送请求
- SendForwardFilter：优先级 500，使用 Servlet 发送请求