---
title: Spring Cloud 微服务（25） --- Zuul(六) <BR> Zuul 文件上传、使用技巧
date: 2019-02-21 16:09:28
updated: 2019-02-21 16:09:28
categories:
    Java
tags: [SpringCloud, Zuul]
---


之前在 `https://www.laiyy.top/java/2019/01-24/spring-cloud-10.html` 介绍了使用 Feign 做文件上传的操作，使用 Zuul 做文件上传，实际上是在 feign 调用之外增加了一层 zuul 路由。

<!-- more -->

# Zuul 文件上传

***源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-zuul/spring-cloud-zuul-file-upload***

## Zuul Server

```yml
server:
  port: 5555
spring:
  application:
    name: spring-cloud-zuul-file-upload
  servlet:
    multipart:
      enabled: true # 使用 http multipart 上传
      max-file-size: 100MB # 文件最大大小，默认 1M，不配置则为 -1
      max-request-size: 100MB # 请求最大大小，默认 10M，不配置为 -1
      file-size-threshold: 1MB # 当上传文件达到 1NB 时进行磁盘写入
      location: / # 上传的临时目录
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
    instance-id: ${spring.application.name}:${server.port}

hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 30000 # 超时时间 30 秒，防止大文件上传出现超时
ribbon:
  ConnectionTimeout: 3000 # Ribbon 链接超时时间
  ReadTimeout: 30000    # Ribbon 读超时时间
```

```java
@SpringBootApplication
@EnableZuulProxy
@EnableDiscoveryClient
@RestController
public class SpringCloudZuulFileUploadApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudZuulFileUploadApplication.class, args);
    }


    @PostMapping(value = "/upload")
    public String uploadFile(@RequestParam(value = "file") MultipartFile file) throws Exception {
        byte[] bytes = file.getBytes();
        File fileToSave = new File(file.getOriginalFilename());
        FileCopyUtils.copy(bytes, fileToSave);
        return fileToSave.getAbsolutePath();
    }

}
```

## 测试结果

![Zuul file upload](/images/spring-cloud/zuul/zuul-file-upload.png)


## 注意事项

如果使用的 cloud 版本是 Finchley 之前的版本，在上传中文名称的文件时，会出现乱码的情况。解决办法：在调用接口上加上 `/zuul` 根节点。如： http://localhost:5555/zuul/upload

需要注意的是，在 @RequestMapping、@PostMapping 上不能加 `/zuul`，这个节点是 zuul 自带的。也就是说，即是在项目中没有 `/zuul` 开头的映射，使用 zuul 后都会加上 `/zuul` 根映射。

---

# Zuul 使用技巧

## Zuul 饥饿加载

Zuul 内部使用 Ribbon 远程调用，根据 Ribbon 的特性，第一次调用会去注册中心获取注册表，初始化 Ribbon 负载信息，这是一种懒加载策略，但是这个过程很耗时。为了避免这个问题，可以使用饥饿加载

```yml
zuul:
  ribbon:
    eager-load:
      enabled: true
```

## 修改请求体

***源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-zuul/spring-cloud-zuul-change-param***

### zuul server

```java
@Configuration
public class ChangeParamZuulFilter extends ZuulFilter {
    @Override
    public String filterType() {
        return FilterConstants.PRE_TYPE;
    }

    @Override
    public int filterOrder() {
        return FilterConstants.PRE_DECORATION_FILTER_ORDER + 1;
    }

    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() throws ZuulException {
        RequestContext currentContext = RequestContext.getCurrentContext();
        Map<String, List<String>> requestQueryParams = currentContext.getRequestQueryParams();
        if (requestQueryParams == null) {
            requestQueryParams = Maps.newHashMap();
        }
        
        List<String> arrayList = Lists.newArrayList();
        // 增加一个参数
        arrayList.add("1111111");
        requestQueryParams.put("test", arrayList);
        currentContext.setRequestQueryParams(requestQueryParams);
        return null;
    }
}
```

### provider
```java
@RestController
public class TestController {
	
	@PostMapping("/change-params")
	public Map<String, Object> modifyRequestEntity (HttpServletRequest request) {
        Map<String, Object> bodyParams = new HashMap<>();
        Enumeration enu = request.getParameterNames();  
        while (enu.hasMoreElements()) {  
	        String paraName = (String)enu.nextElement();  
	        bodyParams.put(paraName, request.getParameter(paraName));
        }
        return bodyParams;
	}
}
```

### 验证

访问 http://localhost:5555/provider/change-params

![change params](/images/spring-cloud/zuul/zuul-change-params.png)


## zuul 中使用 OkHttp

```xml
<dependency>
    <groupId>com.squareup.okhttp3</groupId>
    <artifactId>okhttp</artifactId>
</dependency>
```

```yml
ribbon:
  okhttp:
    enabled: true
  http:
    client:
      enabled: false
```

## zuul 重试

```xml
<!-- retry -->
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
</dependency>
```

```yml
spring:
  cloud:
    loadbalancer:
      retry:
        enabled: true

zuul:
  retryable: true

ribbon:
  ConnectTimeout: 3000
  ReadTimeout: 60000
  MaxAutoRetries: 1 # 对第一次请求的服务的重试次数
  MaxAutoRetriesNextServer: 1 # 要重试的下一个服务的最大数量（不包括第一个服务）
  OkToRetryOnAllOperations: true
```

## header 传递

```java
@Configuration
public class AddHeaderZuulFilter extends ZuulFilter {
    @Override
    public String filterType() {
        return FilterConstants.PRE_TYPE;
    }

    @Override
    public int filterOrder() {
        return FilterConstants.PRE_DECORATION_FILTER_ORDER + 1;
    }

    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() throws ZuulException {
        RequestContext currentContext = RequestContext.getCurrentContext();
        currentContext.addZuulRequestHeader("key", "value");
        return null;
    }
}
```

在下游 `spring-cloud-change-param-provider` 中查看 header 传递
```java
@RestController
public class TestController {

	@PostMapping("/change-params")
	public Map<String, Object> modifyRequestEntity (HttpServletRequest request) {
        Map<String, Object> bodyParams = new HashMap<>();
        Enumeration enu = request.getParameterNames();  
        while (enu.hasMoreElements()) {  
	        String paraName = (String)enu.nextElement();  
	        bodyParams.put(paraName, request.getParameter(paraName));
        }

        Enumeration<String> headerNames = request.getHeaderNames();
        while (headerNames.hasMoreElements()) {
            String header = headerNames.nextElement();
            String value = request.getHeader(header);
            System.out.println(header + " ---> " + value);
        }
        return bodyParams;
	}
}
```

访问 http://localhost:5555/provider/change-params ，查看控制台：
```
user-agent ---> Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/71.0.3578.98 Safari/537.36
cache-control ---> no-cache
origin ---> chrome-extension://fhbjgbiflinjbdggehcddcbncdddomop
postman-token ---> a99dcd1c-7c76-2d7e-8a4c-fa5c2bdb6222
accept ---> */*
accept-encoding ---> gzip, deflate, br
accept-language ---> zh-CN,zh;q=0.9
x-forwarded-host ---> localhost:5555
x-forwarded-proto ---> http
x-forwarded-prefix ---> /provider
x-forwarded-port ---> 5555
x-forwarded-for ---> 0:0:0:0:0:0:0:1
key ---> value          ---------------------- zuul server 中增加的 header
content-type ---> application/x-www-form-urlencoded;charset=UTF-8
content-length ---> 12
host ---> 10.10.10.141:7070
connection ---> Keep-Alive
```


## Zuul 整合 Swagger

### provider 
```xml
<dependencies>
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
  </dependency>

  <dependency>
      <groupId>io.springfox</groupId>
      <artifactId>springfox-swagger2</artifactId>
      <version>2.7.0</version>
  </dependency>
</dependencies>
```

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableSwagger2   // 这个注解必须加，否则解析不到
public class SpringCloudChangeParamProviderApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudChangeParamProviderApplication.class, args);
    }

}
```

### Zuul Server

```xml
<!-- swagger -->
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
    <version>2.9.2</version>
</dependency>
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
    <version>2.9.2</version>
</dependency>
```

```java
/**
 * @author laiyy
 * @date 2019/2/25 15:24
 * @description
 */
@Configuration
@EnableSwagger2 // 这个注解必须加
public class Swagger2Configuration {

    private final ZuulProperties zuulProperties;

    @Autowired
    public Swagger2Configuration(ZuulProperties zuulProperties) {
        this.zuulProperties = zuulProperties;
    }

    @Bean
    public Docket createRestApi() {
        return new Docket(DocumentationType.SWAGGER_2)
                .apiInfo(apiInfo());
    }

    private ApiInfo apiInfo() {
        return new ApiInfoBuilder().title("spring cloud swagger 2")
                .description("spring cloud 整合 swagger2")
                .termsOfServiceUrl("")
                .contact(new Contact("laiyy", "laiyy0728@gmail.com", "laiyy0728@gmail.com")).version("1.0")
                .build();
    }



// 第一种配置方式
    @Primary
    @Bean
    public SwaggerResourcesProvider swaggerResourcesProvider() {
        return () -> {
            List<SwaggerResource> resources = new ArrayList<>();
            zuulProperties.getRoutes().values().stream()
                    .forEach(route -> resources.add(createResource(route.getServiceId(), route.getServiceId(), "2.0")));
            return resources;
        };
    }

    private SwaggerResource createResource(String name, String location, String version) {
        SwaggerResource swaggerResource = new SwaggerResource();
        swaggerResource.setName(name);
        swaggerResource.setLocation("/" + location + "/v2/api-docs");
        swaggerResource.setSwaggerVersion(version);
        return swaggerResource;
    }



// 第二种配置方式（推荐使用）
//    @Component
//    @Primary
//    public class ZuulSwaggerResourceProvider implements SwaggerResourcesProvider {
//
//        private final RouteLocator routeLocator;
//
//        @Autowired
//        public ZuulSwaggerResourceProvider(RouteLocator routeLocator) {
//            this.routeLocator = routeLocator;
//        }
//
//        @Override
//        public List<SwaggerResource> get() {
//            List<SwaggerResource> resources = Lists.newArrayList();
//            routeLocator.getRoutes().forEach(route -> {
//                resources.add(createResource(route.getId(), route.getFullPath().replace("**", "v2/api-docs")));
//            });
//            return resources;
//        }
//
//        private SwaggerResource createResource(String name, String location) {
//            SwaggerResource swaggerResource = new SwaggerResource();
//            swaggerResource.setName(name);
//            swaggerResource.setLocation(location);
////            swaggerResource.setLocation("/" + location + "/api-docs");
//            swaggerResource.setSwaggerVersion("2.0");
//            return swaggerResource;
//        }
//    }
}
```

### 验证

访问 http://localhost:5555/swagger-ui.html 
![Zuul Swagger](/images/spring-cloud/zuul/zuul-swagger.png)