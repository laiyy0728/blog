---
title: Spring Cloud 微服务（20） --- Zuul(二) <br> Zuul Filter 链
date: 2019-02-15 16:03:31
updated: 2019-02-15 16:03:31
categories:
    Java
tags:
    - SpringCloud
    - Zuul
---

Zuul 的核心逻辑是由一系列紧密配合工作的 Filter 来实现的，能够在进行 HTTP 请求或响应的时候执行相关操作。

<!-- more -->

# Zuul Filter

## Zuul Filter 的特点

- Filter 类型：Filter 类型决定了当前的 Filter 在整个 Filter 链中的执行顺序。
- Filter 执行顺序：同一种类型的 Filter 通过 filterOrder() 来设置执行顺序
- Filter 执行条件：Filter 执行所需的标准、条件
- Filter 执行效果：符合某个条件，产生的执行结果

Zuul 内部提供了一个动态读取、编译、运行这些 Filter 的机制。Filter 之间不直接通信，在请求线程中会通过 RequestContext 共享状态，内部使用 ThreadLocal 实现，也可以在 Filter 之间使用 ThreadLocal 收集自己需要的状态、数据

Zuul Filter 的执行逻辑源码在 `com.netflix.zuul.http.ZuulServlet` 中
```java
public void service(ServletRequest servletRequest, ServletResponse servletResponse) throws ServletException, IOException {
    try {
        this.init((HttpServletRequest)servletRequest, (HttpServletResponse)servletResponse);
        RequestContext context = RequestContext.getCurrentContext();        // 通过 RequestContext 获取共享状态
        context.setZuulEngineRan();

        try {
            this.preRoute();        // 执行请求之前的操作
        } catch (ZuulException var13) {
            this.error(var13);      // 出现错误的操作
            this.postRoute();
            return;
        }

        try {
            this.route();       // 路由操作
        } catch (ZuulException var12) {
            this.error(var12);
            this.postRoute();
            return;
        }

        try {
            this.postRoute();       // 请求操作
        } catch (ZuulException var11) {
            this.error(var11);
        }
    } catch (Throwable var14) {
        this.error(new ZuulException(var14, 500, "UNHANDLED_EXCEPTION_" + var14.getClass().getName()));
    } finally {
        RequestContext.getCurrentContext().unset();
    }
}

```


## Zuul 生命周期

Zuul 官方文档中，生命周期图。
![Zuul Life Cycle](/images/spring-cloud/zuul/zuul-life-cycle.png)

但官方文档的生命周期图不太准确。
- 在 postRoute 执行之前，即 postFilter 执行之前，如果没有出现过错误，会调用 error 方法，并调用 this.error(new ZuulException) 打印堆栈信息
- 在 postRoute 执行之前就已经报错，会调用 error 方法，再调用 postRoute，但是之后会直接 return，不会调用 this.error(new ZuulException) 打印堆栈信息

由此可以看出，整个 Filter 调用链的重点可能是 postFilter 也可能是 errorFilter

![Zuul Life Cycle](/images/spring-cloud/zuul/zuul-life-cycle-1.png)

pre、route 出现错误后，进入 error，再进入 post，再返回
pre、route 没有出现错误，进入 post，如果出现错误，再进入 error，再返回

- pre：在 Zuul 按照规则路由到下级服务之前执行。如果需要对请求进行预处理，如：鉴权、限流等，都需要在此 Filter 实现
- route：Zuul 路由动作的执行者，是 Http Client、Ribbon 构建和发送原始 HTTP 请求的地方
- post：源服务返回结果或异常信息发生后执行，如果需要对返回值信息做处理，需要实现此类 Filter
- error：整个生命周期发生异常，都会进入 error Filter，可做全局异常处理。

Filter 之间，通过 `com.netflix.zuul.context.RequestContext` 类进行通信，内部采用 ThreadLocal 保存每个请求的一些信息，包括：请求路由、错误信息、HttpServletRequest、HTTPServletResponse，扩展了 ConcurrentHashMap，目的是为了在处理过程中保存任何形式的信息

---

# Zuul 原生 Filter

整合 `spring-boot-starter-actuator` 后，查看 idea 控制台 endpoints 栏的 mappings，可以看到多了几个 Actuator 端点

## routes 端点

访问 http://localhost:8989/actuator/routes 可以查看当前 zuul server 映射了几个路径、服务
```json
{
    "/provider/**": "spring-cloud-provider-service-simple",
    "/spring-cloud-provider-service-simple/**": "spring-cloud-provider-service-simple"
}
```

访问 http://localhost:8989/actuator/routes/details 可以查看具体的映射信息
```json
{
    "/provider/**": {
        "id": "spring-cloud-provider-service-simple",       // serviceId
        "fullPath": "/provider/**",     // 映射 path
        "location": "spring-cloud-provider-service-simple",     // 服务名称，实际上也是 serviceId
        "path": "/**",      // 实际访问路径 
        "prefix": "/provider",      // 访问前缀
        "retryable": false,     // 是否开启重试
        "customSensitiveHeaders": false,        // 是否自定义了敏感 header
        "prefixStripped": true      // 是否去掉前缀（如果为 false，则实际访问时需要加 前缀，且实际请求的访问路径也会加上前缀）
    },
    "/spring-cloud-provider-service-simple/**": {
        "id": "spring-cloud-provider-service-simple",
        "fullPath": "/spring-cloud-provider-service-simple/**",
        "location": "spring-cloud-provider-service-simple",
        "path": "/**",
        "prefix": "/spring-cloud-provider-service-simple",
        "retryable": false,
        "customSensitiveHeaders": false,
        "prefixStripped": true
    }
}
```

## filters 端点

访问 http://localhost:8989/actuator/filters ，返回当前 zuul 的所有 filters

![Zuul Filters](/images/spring-cloud/zuul/zuul-filters.png)

## 内置 Filters

| 名称 | 类型 | 顺序 | 描述 |
| :-: | :-: | :-: | :-: |
| ServletDetectionFilter | pre | -3 | 通过 Spring Dispatcher 检查请求是否通过 |
| Servlet30WrapperFilter | pre | -2 | 适配 HttpServletRequest 为 Servlet30RequestWrapper 对象 |
| FormBodyWrapperFilter | pre | -1 | 解析表单数据，并为下游请求进行重新编码 |
| DebugFilter | pre | 1 | Debug 路由标识 |
| PreDecorationFilter | pre | 5 | 处理请求上下文供后续使用，设置下游相关头信息 |
| RibbonRoutingFilter | route | 10 | 使用 Ribbon、Hystrix、嵌入式 HTTP 客户端发送请求 |
| SimpleHostRoutingFilter | route | 100 | 使用 Apache Httpclient 发送请求 |
| SendForwardFilter | route | 500 | 使用 Servlet 转发请求 |
| SendResponseFilter | post | 1000 | 将代理请求的响应写入当前响应 |
| SendErrorFilter | error | 0 | 如果 RequestContext.getThrowable() 不为空，则转发到 error.path 哦诶之的路径 |


如果使用 `@EnableZuulServer` 注解，将减少 `PreDecorationFilter`、`RibbonRoutingFilter`、`SimpleHostRoutingFilter`

如果要替换到某个原生的 Filter，可以自实现一个和原生 Filter 名称、类型一样的 Filter，并替换。或者禁用掉某个filter，并自实现一个新的。
禁用语法： `zuul.{SimpleClassName}.{filterType}.disable=true`，如 `zuul.SendErrorFilter.error.disable=true`


---

# 多级业务处理

在 Zuul Filter 链体系中，可以把一组业务逻辑细分，然后封装到一个个紧密结合的 Filter，设置处理顺序，组成一组 Filter 链。

## 自定义实现 Filter

在 Zuul 中实现自定义 Filter，继承 `ZuulFilter` 类即可，ZuulFilter 是一个抽象类，需要实现以下几个方法

- String filterType：使用返回值设定 Filter 类型，可以设置为 `pre`、`route`、`post`、`error`
- int filterOrder：使用返回值设置 Filter 执行次序
- boolean shouldFilter：使用返回值设定该 Filter 是否执行，可以作为开关来使用
- Object run：Filter 的核心执行逻辑

```java
// 自定义 ZuulFilter
public class FirstPreFilter extends ZuulFilter {
    @Override
    public String filterType() {
        return FilterConstants.PRE_TYPE;
    }

    @Override
    public int filterOrder() {
        return 0;
    }

    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() throws ZuulException {
        System.out.println("自定义 Filter，类型为 pre！");
        return null;
    }
}


// 注入 Spring 容器
@Bean
public FirstPreFilter firstPreFilter(){
    return new FirstPreFilter();
}
```

此时访问 http://localhost:8989/provider/get-result ，查看控制台：
```
Initializing Servlet 'dispatcherServlet'
Completed initialization in 0 ms
自定义 Filter，类型为 pre！
Flipping property: spring-cloud-provider-service-simple.ribbon.ActiveConnectionsLimit to use NEXT property: niws.loadbalancer.availabilityFilteringRule.activeConnectionsLimit = 2147483647
Shutdown hook installed for: NFLoadBalancer-PingTimer-spring-cloud-provider-service-simple
```


## 业务处理

使用 SecondFilter 验证是否传入参数 a，ThirdPreFilter 验证是否传入参数 b，在 PostFilter 统一处理返回内容。

***SecondPreFilter***
```java
public class SecondPreFilter extends ZuulFilter {
    private static final Logger LOGGER = LoggerFactory.getLogger(SecondPreFilter.class);
    @Override
    public String filterType() {
        return FilterConstants.PRE_TYPE;
    }

    @Override
    public int filterOrder() {
        return 2;
    }

    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() throws ZuulException {
        LOGGER.info(">>>>>>>>>>>>> SecondPreFilter ！ <<<<<<<<<<<<<<<<");
        // 获取上下文
        RequestContext requestContext = RequestContext.getCurrentContext();
        // 从上下文获取 request
        HttpServletRequest request = requestContext.getRequest();
        // 从 request 获取参数 a
        String a = request.getParameter("a");
        // 如果参数 a 为空
        if (StringUtils.isBlank(a)) {
            LOGGER.info(">>>>>>>>>>>>>>>> 参数 a 为空！ <<<<<<<<<<<<<<<<");
            // 禁止路由，禁止访问下游服务
            requestContext.setSendZuulResponse(false);
            // 设置 responseBody，供 postFilter 使用
            requestContext.setResponseBody("{\"status\": 500, \"message\": \"参数 a 为空！\"}");
            // 用于下游 Filter 判断是否执行
            requestContext.set("logic-is-success", false);
            // Filter 结束
            return null;
        }
        requestContext.set("logic-is-success", true);
        return null;
    }
}
```

***ThirdPreFilter***
```java
public class ThirdPreFilter extends ZuulFilter {

    private static final Logger LOGGER = LoggerFactory.getLogger(ThirdPreFilter.class);

    @Override
    public String filterType() {
        return FilterConstants.PRE_TYPE;
    }

    @Override
    public int filterOrder() {
        return 3;
    }

    @Override
    public boolean shouldFilter() {
        RequestContext context = RequestContext.getCurrentContext();
        // 获取上下文中的 logic-is-success 中的值，用于判断当前 filter 是否执行
        return (boolean) context.get("logic-is-success");
    }

    @Override
    public Object run() throws ZuulException {
        LOGGER.info(">>>>>>>>>>>>> ThirdPreFilter ！ <<<<<<<<<<<<<<<<");
        // 获取上下文
        RequestContext requestContext = RequestContext.getCurrentContext();
        // 从上下文获取 request
        HttpServletRequest request = requestContext.getRequest();
        // 从 request 获取参数 a
        String a = request.getParameter("b");
        // 如果参数 a 为空
        if (StringUtils.isBlank(a)) {
            LOGGER.info(">>>>>>>>>>>>>>>> 参数 b 为空！ <<<<<<<<<<<<<<<<");
            // 禁止路由，禁止访问下游服务
            requestContext.setSendZuulResponse(false);
            // 设置 responseBody，供 postFilter 使用
            requestContext.setResponseBody("{\"status\": 500, \"message\": \"参数 b 为空！\"}");
            // 用于下游 Filter 判断是否执行
            requestContext.set("logic-is-success", false);
            // Filter 结束
            return null;
        }
        requestContext.set("logic-is-success", true);
        return null;
    }
}
```

***PostFilter***
```java

public class PostFilter extends ZuulFilter {

    private static final Logger LOGGER = LoggerFactory.getLogger(PostFilter.class);

    @Override
    public String filterType() {
        return FilterConstants.POST_TYPE;
    }

    @Override
    public int filterOrder() {
        return 0;
    }

    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() throws ZuulException {
        LOGGER.info(">>>>>>>>>>>>>>>>>>> Post Filter! <<<<<<<<<<<<<<<<");
        RequestContext context = RequestContext.getCurrentContext();
        // 处理返回中文乱码
        context.getResponse().setCharacterEncoding("UTF-8");
        // 获取上下文保存的 responseBody
        String responseBody = context.getResponseBody();
        // 如果 responseBody 不为空，则证明流程中有异常发生
        if (StringUtils.isNotBlank(responseBody)) {
            // 设置返回状态码
            context.setResponseStatusCode(500);
            // 替换响应报文
            context.setResponseBody(responseBody);
        }
        return null;
    }
}
```

访问 http://localhost:8989/provider/add 、http://localhost:8989/provider/add?a=1 、http://localhost:8989/provider/add?a=1&b=1 ，查看控制台

控制台：
```
2019-02-18 14:09:44.890  INFO 5800 --- [nio-8989-exec-7] c.l.g.z.s.filter.FirstPreFilter          : >>>>>>>>>>>>>>>>> 自定义 Filter，类型为 pre！ <<<<<<<<<<<<<<<<<<
2019-02-18 14:09:44.890  INFO 5800 --- [nio-8989-exec-7] c.l.g.z.s.filter.SecondPreFilter         : >>>>>>>>>>>>> SecondPreFilter ！ <<<<<<<<<<<<<<<<
2019-02-18 14:09:44.890  INFO 5800 --- [nio-8989-exec-7] c.l.g.z.s.filter.SecondPreFilter         : >>>>>>>>>>>>>>>> 参数 a 为空！ <<<<<<<<<<<<<<<<
2019-02-18 14:09:44.890  INFO 5800 --- [nio-8989-exec-7] c.l.g.z.s.filter.PostFilter              : >>>>>>>>>>>>>>>>>>> Post Filter! <<<<<<<<<<<<<<<<
```
```
2019-02-18 14:10:13.004  INFO 5800 --- [nio-8989-exec-5] c.l.g.z.s.filter.FirstPreFilter          : >>>>>>>>>>>>>>>>> 自定义 Filter，类型为 pre！ <<<<<<<<<<<<<<<<<<
2019-02-18 14:10:13.004  INFO 5800 --- [nio-8989-exec-5] c.l.g.z.s.filter.SecondPreFilter         : >>>>>>>>>>>>> SecondPreFilter ！ <<<<<<<<<<<<<<<<
2019-02-18 14:10:13.004  INFO 5800 --- [nio-8989-exec-5] c.l.g.z.s.filter.ThirdPreFilter          : >>>>>>>>>>>>> ThirdPreFilter ！ <<<<<<<<<<<<<<<<
2019-02-18 14:10:13.004  INFO 5800 --- [nio-8989-exec-5] c.l.g.z.s.filter.ThirdPreFilter          : >>>>>>>>>>>>>>>> 参数 b 为空！ <<<<<<<<<<<<<<<<
2019-02-18 14:10:13.005  INFO 5800 --- [nio-8989-exec-5] c.l.g.z.s.filter.PostFilter              : >>>>>>>>>>>>>>>>>>> Post Filter! <<<<<<<<<<<<<<<<
```
```
2019-02-18 14:10:28.488  INFO 5800 --- [nio-8989-exec-9] c.l.g.z.s.filter.FirstPreFilter          : >>>>>>>>>>>>>>>>> 自定义 Filter，类型为 pre！ <<<<<<<<<<<<<<<<<<
2019-02-18 14:10:28.488  INFO 5800 --- [nio-8989-exec-9] c.l.g.z.s.filter.SecondPreFilter         : >>>>>>>>>>>>> SecondPreFilter ！ <<<<<<<<<<<<<<<<
2019-02-18 14:10:28.488  INFO 5800 --- [nio-8989-exec-9] c.l.g.z.s.filter.ThirdPreFilter          : >>>>>>>>>>>>> ThirdPreFilter ！ <<<<<<<<<<<<<<<<
2019-02-18 14:10:28.500  INFO 5800 --- [nio-8989-exec-9] c.l.g.z.s.filter.PostFilter              : >>>>>>>>>>>>>>>>>>> Post Filter! <<<<<<<<<<<<<<<<
```

返回值：
```json
{"status": 500, "message": "参数 a 为空！"}
```
```json
{"status": 500, "message": "参数 b 为空！"}
```
```
result is : a + b = 2
```

由此验证自定义 Zuul Filter 成功。