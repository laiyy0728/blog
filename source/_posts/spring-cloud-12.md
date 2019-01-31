---
title: Spring Cloud 微服务（12） --- Ribbon(二) <BR> 工作原理
date: 2019-01-31 10:18:14
updated: 2019-01-31 10:18:14
categories:
    Java
tags:
    - SpringCloud
    - Ribbon
---


# Ribbon 核心接口

| 接口 | 描述 | 默认实现类 |
| :-: | :-: | :-: |
| IClientConfig | 定义 Ribbon 中管理配置的接口 | DefaultClientConfigImpl |
| IRule | 定义 Ribbon 中负载均衡策略的接口 | RoundRobinRule | 
| IPing | 定义定期 ping 服务检查可用性的接口 | DummyPing | 
| ServerList<Server> | 定义获取服务列表方法的接口 | ConfigurationBasedServerList |
| ServerListFilter<Server> | 定义特定期望获取服务列表方法的接口 | ZonePreferenceServerListFilter |
| ILoadBalanacer | 定义负载均衡选择服务的核心方法的接口 | BaseLoadBalancer | 
| ServerListUpdater | 为 DynamicServerListLoadBalancer 定义动态更新服务列表的接口 | PollingServerListUpdater |

<!-- more -->

---

# Ribbon 的运行原理

Ribbon 实现负载均衡，基本用法是注入一个 RestTemplate，并在 RestTemplate 上使用 @LoadBalanced，才能使 RestTemplate 具备负载均衡能力。

## @LoadBalanced

```java
/**
 * Annotation to mark a RestTemplate bean to be configured to use a LoadBalancerClient
 * @author Spencer Gibb
 */
@Target({ ElementType.FIELD, ElementType.PARAMETER, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Qualifier
public @interface LoadBalanced {
}
```

这个注解标注一个 RestTemplate，使用 LoadBalancerClient，那么 `LoadBalancerClient` 又是什么？

## LoadBalancerClient

```java
/**
 * Represents a client side load balancer
 * @author Spencer Gibb
 */
public interface LoadBalancerClient extends ServiceInstanceChooser {

	<T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException;

	<T> T execute(String serviceId, ServiceInstance serviceInstance, LoadBalancerRequest<T> request) throws IOException;
	
	URI reconstructURI(ServiceInstance instance, URI original);
}
```

LoadBalancerClient 又扩展了 `ServiceInstanceChooser` 接口
```java
public interface ServiceInstanceChooser {

    ServiceInstance choose(String serviceId);
}
```

## 方法解释

- `ServiceInstance choose(String serviceId)`：根据 serviceId，结合负载均衡器，选择一个服务实例
- `<T> T execute(String serviceId, LoadBalancerRequest<T> request)`：使用 LoadBalancer 的 serviceInstance 为置顶的服务执行请求
- `<T> T execute(String serviceId, ServiceInstance serviceInstance, LoadBalancerRequest<T> request)`：使用来自 LoadBalancer 的 ServiceInstance 为指定的服务执行请求，是上一个方法的重载，是上一个方法的细节实现
- `URI reconstructURI(ServiceInstance instance, URI original)`：使用主机 ip、port 构建特定的 URI，供 RIbbon 内部使用。Ribbon 使用服务名称的 URI 作为host。如：http://instance-id/path/to/service

## LoadBalancer 初始化

`LoadBalancerAutoConfiguration` 是 Ribbon 负载均衡初始化加载类，启动的关键核心代码如下：

```java
@Configuration
@ConditionalOnClass(RestTemplate.class)
@ConditionalOnBean(LoadBalancerClient.class)
@EnableConfigurationProperties(LoadBalancerRetryProperties.class)
public class LoadBalancerAutoConfiguration {
    @Bean
	@ConditionalOnMissingBean
	public LoadBalancerRequestFactory loadBalancerRequestFactory(
			LoadBalancerClient loadBalancerClient) {
		return new LoadBalancerRequestFactory(loadBalancerClient, transformers);
	}

	@Configuration
	@ConditionalOnMissingClass("org.springframework.retry.support.RetryTemplate")
	static class LoadBalancerInterceptorConfig {
		@Bean
		public LoadBalancerInterceptor ribbonInterceptor(
				LoadBalancerClient loadBalancerClient,
				LoadBalancerRequestFactory requestFactory) {
			return new LoadBalancerInterceptor(loadBalancerClient, requestFactory);
		}

		@Bean
		@ConditionalOnMissingBean
		public RestTemplateCustomizer restTemplateCustomizer(
				final LoadBalancerInterceptor loadBalancerInterceptor) {
			return restTemplate -> {
                List<ClientHttpRequestInterceptor> list = new ArrayList<>(
                        restTemplate.getInterceptors());
                list.add(loadBalancerInterceptor);
                restTemplate.setInterceptors(list);
            };
		}
	}
}
```

可以看到，在类注解上，`@ConditionalOnClass(RestTemplate.class)`、`@ConditionalOnBean(LoadBalancerClient.class)`，必须在当前工程下有 `RestTemplate` 的实例、必须已经初始化了 `LoadBalancerClient` 的实现类，才会加载 LoadBalancer 的自动装配。

其中 `LoadBalancerRequestFactory` 用于创建 `LoadBalancerRequest`，以供 `LoadBalancerInterceptor` 使用(在低版本没有)，`LoadBalancerInterceptorConfig` 中维护了 `LoadBalancerInterceptor`、`RestTemplateCustomizer` 的实例。

- LoadBalancerInterceptor：拦截每一次 HTTP 请求，将请求绑定进 Ribbon 负载均衡的生命周期
- RestTemplateCustomizer：为每个 RestTemplate 绑定 LoadBalancerInterceptor 拦截器

## LoadBalancerInterceptor

```java
public class LoadBalancerInterceptor implements ClientHttpRequestInterceptor {

	private LoadBalancerClient loadBalancer;
	private LoadBalancerRequestFactory requestFactory;

	public LoadBalancerInterceptor(LoadBalancerClient loadBalancer, LoadBalancerRequestFactory requestFactory) {
		this.loadBalancer = loadBalancer;
		this.requestFactory = requestFactory;
	}

	public LoadBalancerInterceptor(LoadBalancerClient loadBalancer) {
		// for backwards compatibility
		this(loadBalancer, new LoadBalancerRequestFactory(loadBalancer));
	}

	@Override
	public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
			final ClientHttpRequestExecution execution) throws IOException {
		final URI originalUri = request.getURI();
		String serviceName = originalUri.getHost();
		Assert.state(serviceName != null, "Request URI does not contain a valid hostname: " + originalUri);
		return this.loadBalancer.execute(serviceName, requestFactory.createRequest(request, body, execution));
	}
}
```

LoadBalancerInterceptor 利用 `ClientHttpRequestInterceptor` 对每次 HTTP 请求进行拦截，这个类是 Spring 中维护的请求拦截器。可以看到，拦截的请求使用了 `LoadBalancerClient` 的 execute 方法处理请求(由于 RestTemplate 中使用服务名当做 host，所以此时 getHosts() 获取到的服务名)，LoadBalancerClient 只有一个实现类：`RibbonLoadBalancerClient`，具体是 execute 方法如下：

```java
@Override
public <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException {
    ILoadBalancer loadBalancer = getLoadBalancer(serviceId);
    Server server = getServer(loadBalancer);
    if (server == null) {
        throw new IllegalStateException("No instances available for " + serviceId);
    }
    RibbonServer ribbonServer = new RibbonServer(serviceId, server, isSecure(server,
            serviceId), serverIntrospector(serviceId).getMetadata(server));

    return execute(serviceId, ribbonServer, request);
}
```

可以看到，源码中首先获取一个 LoadBalancer，再去获取一个 Server，那么，这个 Server 就是具体服务实例的封装了。既然 Server 是一个具体的服务实例，那么， getServer(loadBalancer) 就是发生负载均衡的地方。

```java
protected Server getServer(ILoadBalancer loadBalancer) {
    if (loadBalancer == null) {
        return null;
    }
    return loadBalancer.chooseServer("default"); // TODO: better handling of key
}
```

查看 chooseServer 方法具体实现（BaseLoadBalancer）：
```java
public Server chooseServer(Object key) {
    if (counter == null) {
        counter = createCounter();
    }
    counter.increment();
    if (rule == null) {
        return null;
    } else {
        try {
            return rule.choose(key);
        } catch (Exception e) {
            logger.warn("LoadBalancer [{}]:  Error choosing server for key {}", name, key, e);
            return null;
        }
    }
}
```

rule.choose(key) 的中的 rule，就是 IRule，而 IRule 就是 Ribbon 的负载均衡策略。由此可以证明，HTTP 请求域负载均衡策略关联起来了。


---

# IRule

IRule 源码：
```java
public interface IRule{
    /*
     * choose one alive server from lb.allServers or
     * lb.upServers according to key
     * 
     * @return choosen Server object. NULL is returned if none
     *  server is available 
     */

    public Server choose(Object key);
    
    public void setLoadBalancer(ILoadBalancer lb);
    
    public ILoadBalancer getLoadBalancer();    
}
```

IRule 中一共定义了 3 个方法，实现类实现 choose 方法，会加入具体的负载均衡策略逻辑，另外两个方法与 ILoadBalancer 关联起来。
在调用过程中，Ribbon 通过 ILoadBalancer 关联 IRule，ILoadBalancer 的 chooseServer 方法会转为为调用 IRRule 的 choose 方法，抽象类 `AbstractLoadBalancerRule` 实现了这两个方法，从而将 ILoadBalancer 与 IRule 关联起来。