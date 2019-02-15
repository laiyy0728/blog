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