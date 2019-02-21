---
title: Spring Cloud 微服务（23） --- Zuul(四) <BR>限流、动态路由概念
date: 2019-02-19 16:40:14
updated: 2019-02-19 16:40:14
categories:
    Java
tags: [SpringCloud, Zuul]
---

之前利用 Hystrix，通过熔断器实现了`通过某个阈值来对异常流量进行降级处理`。除了对异常流量进行降级之外，还可以通过 `流量排队`、`限流`、`分流`等操作，防止系统出错。

<!-- more -->

# 限流算法

限流算法一般分为 `漏桶`、`令牌桶` 两种。

## 漏桶

漏桶的圆形是一个底部有漏孔的桶，桶的上方有一个入水口，水不断流进桶内，桶下方的漏孔会以一个相对恒定的速度漏水，在`入大于出`的情况下，桶在一段时间内就会被装满，这时候多余的水就会溢出；而在`入小于出`的情况下，漏桶起不到任何作用。

当请求或者具有一定体量的数据进入系统时，在漏桶作用下，流量被整形，不能满足要求的部分被削减掉，漏桶算法能强制限定流量速度。溢出的流量可以被再次利用起来，并非完全丢弃，可以把溢出的流量收集到一个队列中，做流量排队，尽量合理利用所有资。
![leaky-bucket](/images/spring-cloud/zuul/leaky-bucket.png)

## 令牌桶

令牌桶与漏桶的区别是，桶里放的是令牌而不是流量，令牌以一个恒定的速度被加入桶内，可以积压，可以溢出。当流量涌入时，量化请求用于获取令牌，如果取到令牌则方形，同时桶内丢掉这个令牌；如果取不到令牌，则请求被丢弃。
由于桶内可以存一定量的令牌，那么就可能会解决一定程度的流量突发。这个也是漏桶与令牌桶的适用场景不同之处。

![token-bucket](/images/spring-cloud/zuul/token-bucket.png)

---

# 限流实例

在 Zuul 中实现限流，最简单的方式是使用 Filter 加上相关的限流算法，其中可能会考虑到 Zuul 多节点部署。因为算法的原因，这是需要一个 K/V 存储工具（Redis等）。

[`spring-cloud-zuul-ratelimit`](https://github.com/marcosbarbero/spring-cloud-zuul-ratelimit) 是一个针对 Zuul 的限流库
限流粒度的策略：
- user：认证用户名或匿名，针对某用户粒度进行限流
- origin：客户机 ip，针对请求客户机 ip 粒度进行限流
- url：特定 url，针对某个请求 url 粒度进行限流
- serviceId：特定服务，针对某个服务 id 粒度进行限流

限流粒度临时变量存储方式：
- IN_MEMORY：基于本地内存，底层是 ConcurrentHashMap
- REDIS：基于 Redis K/V 存储
- CONSUL：基于 Consul K/V 存储
- JPA：基于 SpringData JPA，数据库存储
- BUKET4J：使用 Java 编写的基于令牌桶算法的限流库，四种模：`JCache`、`Hazelcast`、`Apache Ignite`、`Inifinispan`，后面三种支持异步

***源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-zuul/spring-cloud-zuul-ratelimit***

## Zuul Server

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
    </dependency>
    <dependency>
        <groupId>com.marcosbarbero.cloud</groupId>
        <artifactId>spring-cloud-zuul-ratelimit</artifactId>
        <version>2.0.6.RELEASE</version>
    </dependency>
</dependencies>
```

```yml
server:
  port: 5555
spring:
  application:
    name: spring-cloud-ratelimit-zuul-server
eureka:
  instance:
    instance-id: ${spring.application.name}:${server.port}
    prefer-ip-address: true
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
zuul:
  routes:
    spring-cloud-ratelimit-provider-service:
      path: /provider/**
      serviceId: spring-cloud-ratelimit-provider-service
  ratelimit:
    key-prefix: springcloud # 按粒度拆分的临时变量 key 的前缀
    enabled: true # 启用开关
    repository: in_memory # key 的存储类型，默认是 in_memory
    behind-proxy: true # 表示代理之后
    default-policy:
      limit: 2 # 在一个单位时间内的请求数量
      quota: 1 # 在一个单位时间内的请求时间限制
      refresh-interval: 3 # 单位时间窗口
      type: 
        - user # 可指定用户粒度
        - origin # 可指定客户端地址粒度
        - url # 可指定 url 粒度
```

## Provider

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
</dependencies>
```

```yml
spring:
  application:
    name: spring-cloud-ratelimit-provider-service
server:
  port: 7070
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
    instance-id: ${spring.application.name}:${server.port}
```

## 验证

快速访问几次 http://localhost:5555/provider/get-result ，返回值如下：
```json
{
    "timestamp": "2019-02-20T06:52:51.220+0000",
    "status": 429,
    "error": "Too Many Requests",
    "message": "429"
}
```

控制台打印异常如下：
```
com.netflix.zuul.exception.ZuulException: 429
	at com.marcosbarbero.cloud.autoconfigure.zuul.ratelimit.support.RateLimitExceededException.<init>(RateLimitExceededException.java:13) ~[spring-cloud-zuul-ratelimit-core-2.0.6.RELEASE.jar:2.0.6.RELEASE]
	at com.marcosbarbero.cloud.autoconfigure.zuul.ratelimit.filters.RateLimitPreFilter.lambda$run$0(RateLimitPreFilter.java:106) ~[spring-cloud-zuul-ratelimit-core-2.0.6.RELEASE.jar:2.0.6.RELEASE]
	at java.util.ArrayList.forEach(ArrayList.java:1257) ~[na:1.8.0_171]
	at com.marcosbarbero.cloud.autoconfigure.zuul.ratelimit.filters.RateLimitPreFilter.run(RateLimitPreFilter.java:79) ~[spring-cloud-zuul-ratelimit-core-2.0.6.RELEASE.jar:2.0.6.RELEASE]
	at com.netflix.zuul.ZuulFilter.runFilter(ZuulFilter.java:117) ~[zuul-core-1.3.1.jar:1.3.1]
	at com.netflix.zuul.FilterProcessor.processZuulFilter(FilterProcessor.java:193) ~[zuul-core-1.3.1.jar:1.3.1]
	at com.netflix.zuul.FilterProcessor.runFilters(FilterProcessor.java:157) ~[zuul-core-1.3.1.jar:1.3.1]
    ...
```

正常访问结果如下：
```
zuul rate limit result !
```

---

# 动态路由

之前配置路由映射规则的方式，为“静态路由”。如果在迭代过程中，可能需要动态将路由映射规则写入内存。在“静态路由”配置中，需要重启 Zuul 应用。
不需要重启 Zuul，又能修改映射规则的方式，称为“动态路由”。

- SpringCloud Config + Bus，动态刷新配置文件。好处是不用 Zuul 维护映射规则，可以随时修改，随时生效。缺点是需要单独集成一些使用并不频繁的组件。SpringCloud Config 没有可视化界面，维护也麻烦
- 重写 Zuul 配置读取方式，采用事件刷新机制，从数据库读取路由映射规则。此方式基于数据库，可轻松实现管理页面，灵活度高。

# 动态路由实现原理

![动态路由原理核心类依赖图](/images/spring-cloud/zuul/dynamic-route.png)

## DiscoveryClientRouteLocator

```java
public class DiscoveryClientRouteLocator extends SimpleRouteLocator implements RefreshableRouteLocator {
    
    // 省略其他方法

    // 路由
    protected LinkedHashMap<String, ZuulRoute> locateRoutes() {
        // 省略方法实现
    }

    // 刷新
    public void refresh() {
        this.doRefresh();
    }
}
```

`locateRoutes` 方法继承自 `SimpleRouteLocator` 类，并重写规则，该方法主要的功能就是将配置文件中的映射规则信息包装成 `LinkedHashMap<String, ZuulRoute>`，键为路径 path，值 ZuulRoute 是配置文件的封装类。之前的映射配置信息就是使用 ZuulRoute 封装的。
`refresh` 实现自 RefreshableRouteLocator 接口，添加刷新功能必须实现此方法，`doRefresh` 方法来自 `SimpleRouteLocator` 类

## SimpleRouteLocator

`SimpleRouteLocator` 是 `DiscoveryClientRouteLocator` 的父类，此类基本实现了 RouteLocator 接口，对读取配置文件信息做一些处理，提供方法 `doRefresh`、`locateRoutes` 供子类实现刷新策略与映射规则加载策略

```java
/**
 * Calculate all the routes and set up a cache for the values. Subclasses can call
 * this method if they need to implement {@link RefreshableRouteLocator}.
 */
protected void doRefresh() {
    this.routes.set(locateRoutes());
}

/**
 * Compute a map of path pattern to route. The default is just a static map from the
 * {@link ZuulProperties}, but subclasses can add dynamic calculations.
 */
protected Map<String, ZuulRoute> locateRoutes() {
    LinkedHashMap<String, ZuulRoute> routesMap = new LinkedHashMap<>();
    for (ZuulRoute route : this.properties.getRoutes().values()) {
        routesMap.put(route.getPath(), route);
    }
    return routesMap;
}
```

这两个方法都是 protectted 修饰，是为了让子类不用维护此类一些成员变量就能实现刷新或读取路由的功能。从注释上可以看到，调用 `doRedresh` 方法需要实现 `RefreshableRouteLocator`；`locateRoutes` 默认是一个静态的映射读取方法，如果需要动态记载映射，需要子类重写此方法。

## ZuulServerAutoConfiguration

ZuulServerAutoConfiguration 是 Spring Cloud Zuul 的配置类，主要目的是注册各种过滤器、监听器以及其他功能。Zuul 在注册中心新增服务后刷新监听器也是在这个类中注册的，底层是 Spring 的 ApplicationListener

```java
@Configuration
@EnableConfigurationProperties({ ZuulProperties.class })
@ConditionalOnClass(ZuulServlet.class)
@ConditionalOnBean(ZuulServerMarkerConfiguration.Marker.class)
public class ZuulServerAutoConfiguration {

    // 省略其他功能注册

    // Zuul 刷新监听器
    private static class ZuulRefreshListener
			implements ApplicationListener<ApplicationEvent> {

		@Autowired
		private ZuulHandlerMapping zuulHandlerMapping;

		private HeartbeatMonitor heartbeatMonitor = new HeartbeatMonitor();

		@Override
		public void onApplicationEvent(ApplicationEvent event) {
			if (event instanceof ContextRefreshedEvent
					|| event instanceof RefreshScopeRefreshedEvent
					|| event instanceof RoutesRefreshedEvent
					|| event instanceof InstanceRegisteredEvent) {
				reset();
			}
			else if (event instanceof ParentHeartbeatEvent) {
				ParentHeartbeatEvent e = (ParentHeartbeatEvent) event;
				resetIfNeeded(e.getValue());
			}
			else if (event instanceof HeartbeatEvent) {
				HeartbeatEvent e = (HeartbeatEvent) event;
				resetIfNeeded(e.getValue());
			}
		}

		private void resetIfNeeded(Object value) {
			if (this.heartbeatMonitor.update(value)) {
				reset();
			}
		}

		private void reset() {
			this.zuulHandlerMapping.setDirty(true);
		}
	}
}
```

其中，由方法 `onApplicationEvent`可知，Zuul 会接收 4 种事件通知 `ContextRefreshedEvent`、`RefreshScopeRefreshedEvent`、`RoutesRefreshedEvent`、`InstanceRegisteredEvent`，这四种通知都会去刷新路由映射配置信息，此外，心跳续约监视器 `HeartbeatEvent` 也会触发这个动作

## ZuulHandlerMapping

在 `ZuulServerAutoConfiguration#ZuulRefreshListener` 中，注入了 `ZuulHandlerMapping`，此类是将本地配置的映射关系，映射到远程的过程控制器

```java
/**
 * MVC HandlerMapping that maps incoming request paths to remote services.
 */
public class ZuulHandlerMapping extends AbstractUrlHandlerMapping {

    // 省略其他配置

    private volatile boolean dirty = true;

    public void setDirty(boolean dirty) {
		this.dirty = dirty;
		if (this.routeLocator instanceof RefreshableRouteLocator) {
			((RefreshableRouteLocator) this.routeLocator).refresh();
		}
    }
}
```

`dirty` 属性很重要，它是用来控制当前是否需要重新加载映射配置信息的标记，在 Zuul 每次进行路由操作的时候都会检查这个值。如果为 true，则会触发配置信息的重新加载，同时再将其审核制为 false。由 `setDirty` 方法体可知，启动刷新动作必须实现 `RefreshableRouteLocator`，否则会出现类转换异常。