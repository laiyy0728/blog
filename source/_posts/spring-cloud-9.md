---
title: Spring Cloud 微服务（9） --- Feign(三) <BR> http client 替换、GET 方式传递 POJO等
date: 2019-01-23 16:46:48
updated: 2019-01-23 16:46:48
categories:
    Java
tags:
    - SpringCloud
    - Feign
---

在了解了 FeignClient 的配置、请求响应的压缩后，基本的调用已经没有问题。
接下来就需要了解 Feign 多参数传递、文件上传、header 传递 token、请求失败、图片流 等问题的解决，以及 HTTP Client 替换的问题。

<!-- more -->

# Http Client 替换

***源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-feign/spring-cloud-feign-httpclient***

Feign 默认情况下使用的是 JDK 原生的 URLConnection 发送 HTTP 请求，没有连接池，但是对每个地址都会保持一个长连接。可以利用 Apache HTTP Client 替换原始的 URLConnection，通过设置连接池、超时时间等，对服务调用进行调优。

在类 `feign/Client$Default.java` 中，可以看到，默认执行 http 请求的是 URLConnection
```java
public static class Default implements Client {

    @Override
    public Response execute(Request request, Options options) throws IOException {
      HttpURLConnection connection = convertAndSend(request, options);
      return convertResponse(connection).toBuilder().request(request).build();
    }
}
```

在类 `org/springframework/cloud/openfeign/ribbon/FeignRibbonClientAutoConfiguration.java` 中，可以看到引入了三个类：`HttpClientFeignLoadBalancedConfiguration`、`OkHttpFeignLoadBalancedConfiguration`、`DefaultFeignLoadBalancedConfiguration`

可以看到在 `DefaultFeignLoadBalancedConfiguration` 中，使用的是 `Client.Default`，即使用 URLConnection


## 使用 Apache Http Client 替换 URLConnection

### pom 依赖

```xml
<!-- 引入 httpclient -->
<dependency>
    <groupId>org.apache.httpcomponents</groupId>
    <artifactId>httpclient</artifactId>
</dependency>

<!-- 引入 feign 对 httpclient 的支持 -->
<dependency>
    <groupId>com.netflix.feign</groupId>
    <artifactId>feign-httpclient</artifactId>
    <version>8.18.0</version>
</dependency>
```

### 配置文件

```yml
feign:
  httpclient:
    enabled: true
```

### 查看验证配置

在类 `HttpClientFeignLoadBalancedConfiguration` 上，有注解：`@ConditionalOnClass(ApacheHttpClient.class)`、`@ConditionalOnProperty(value = "feign.httpclient.enabled", matchIfMissing = true)`：在 `ApacheHttpClient` 类存在且 `feign.httpclient.enabled` 为 true 时启用配置。

在 `HttpClientFeignLoadBalancedConfiguration` 123 行打上断点，重新启动项目，可以看到确实进行了 ApacheHttpClient 的声明。在将 `feign.httpclient.enabled` 设置为 false 后，断点就进不来了。由此可以验证 ApacheHttpClient 替换成功。

## 使用 OkHttp 替换 URLConnection

### pom 依赖

```xml
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-okhttp</artifactId>
    <version>10.1.0</version>
</dependency>
```

### 配置文件

```yml
feign:
  httpclient:
    enabled: false
  okhttp:
    enabled: true
```

### 验证配置

在 `OkHttpFeignLoadBalancedConfiguration` 第 84 行打断点，重新启动项目，可以看到成功进入断点；当把 `feign.okhttp.enabled` 设置为 false 后，重新启动项目，没进入断点。证明 OkHttp 替换成功。

---

# GET 方式传递 POJO等

***源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-feign/spring-cloud-feign-multi-params***

SpringMVC 是支持 GET 方法直接绑定 POJI 的，但是 Feign 的实现并未覆盖所有 SpringMVC 的功能，常用的解决方式：
- 把 POJO 拆散成一个一个单独的属性放在方法参数里
- 把方法参数变成 Map 传递
- 使用 GET 传递 @RequestBody，这种方式有违 RESTFul。


实现 Feign 的 RequestInterceptor 中的 apply 方法，统一拦截转换处理 Feign 中 GET 方法传递 POJO 问题。而 Feign 进行 POST 多参数传递要比 Get 简单。

## provider

provider 用于模拟用户查询、修改操作，作为服务生产者

### pom 依赖：
```xml
 <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>

<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
</dependency>
```

### 配置文件：
```yml
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    instance-id: ${spring.application.name}:${server.port}
spring:
  application:
    name: spring-cloud-feign-multi-params-provider
server:
  port: 8888
```

### 实体、启动类、Controller
```java

// 实体
@Data
@NoArgsConstructor
@AllArgsConstructor
public class User {

    private int id;

    private String name;

}

// 启动类
@SpringBootApplication
@EnableDiscoveryClient
public class SpringCloudFeignMultiParamsProviderApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudFeignMultiParamsProviderApplication.class, args);
    }

}


// Controller
@RestController
@RequestMapping(value = "/user")
public class UserController {

    @GetMapping(value = "/add")
    public String addUser(User user){
        return "hello!" + user.getName();
    }

    @PostMapping(value = "/update")
    public String updateUser(@RequestBody User user){
        return "hello! modifying " + user.getName();
    }

}
```

## consumer

consumer 用于模拟服务调用，属于服务消费者，调用 provider 的具体实现

### pom 依赖：
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>

<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
</dependency>
```

### 配置文件：
```yml
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    instance-id: ${spring.application.name}:${server.port}
spring:
  application:
    name: spring-cloud-feign-multi-params-consumer
server:
  port: 8889

feign:
  client:
    config:
      spring-cloud-feign-multi-params-provider:
        loggerLevel: full
logging:
  level:
    com.laiyy.gitee.feign.multi.params.springcloudfeignmultiparamscomsumer.MultiParamsProviderFeignClient: debug
```


### 实体、启动类、Controller、FeignClient
```java

// 实体与 provider 一致，不再赘述

// 启动类
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients
public class SpringCloudFeignMultiParamsComsumerApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudFeignMultiParamsComsumerApplication.class, args);
    }

}


// Controller
@RestController
public class UserController {

    private final MultiParamsProviderFeignClient feignClient;

    @Autowired
    public UserController(MultiParamsProviderFeignClient feignClient) {
        this.feignClient = feignClient;
    }

    @GetMapping(value = "add-user")
    public String addUser(User user){
        return feignClient.addUser(user);
    }

    @PostMapping(value = "update-user")
    public String updateUser(@RequestBody User user){
        return feignClient.updateUser(user);
    }

}

// FeignClient
@FeignClient(name = "spring-cloud-feign-multi-params-provider")
public interface MultiParamsProviderFeignClient {

    /**
     * GET 方式
     * @param user user
     * @return 添加结果
     */
    @RequestMapping(value = "/user/add", method = RequestMethod.GET)
    String addUser(User user);

    /**
     * POST 方式
     * @param user user
     * @return 修改结果
     */
    @RequestMapping(value = "/user/update", method = RequestMethod.POST)
    String updateUser(@RequestBody User user);
}
```

## 验证调用

使用 POST MAN 测试工具，调用 consumer 接口，利用 Feign 进行远程调用

调用 `update-user`，验证调用成功

![POST 方式调用 update](/images/spring-cloud/feign/feign-multi-params-update.png)


调用 `add-user`，验证调用失败

![GET 方式调用 add](/images/spring-cloud/feign/feign-multi-params-get-user.png)

控制台报错：
```
{"timestamp":"2019-01-24T08:24:42.887+0000","status":405,"error":"Method Not Allowed","message":"Request method 'POST' not supported","path":"/user/add"}] with root cause

feign.FeignException: status 405 reading MultiParamsProviderFeignClient#addUser(User); content:
{"timestamp":"2019-01-24T08:24:42.887+0000","status":405,"error":"Method Not Allowed","message":"Request method 'POST' not supported","path":"/user/add"}
	at feign.FeignException.errorStatus(FeignException.java:62) ~[feign-core-9.5.1.jar:na]
	at feign.codec.ErrorDecoder$Default.decode(ErrorDecoder.java:91) ~[feign-core-9.5.1.jar:na]
	at feign.SynchronousMethodHandler.executeAndDecode(SynchronousMethodHandler.java:138) ~[feign-core-9.5.1.jar:na]
	...
```

命名是 GET 调用，为什么到底层就变成了 POST 调用？

## GET 传递 POJO 解决方案

Feign 的远程调用中，GET 是不能传递 POJO 的，否则就是 POST，为了解决这个错误，可以实现 RequestInterceptor，解析 POJO，传递 Map 即可解决

在 consumer 中，增加一个实体类，用于解析 POJO

```java
/**
 * @author laiyy
 * @date 2019/1/24 10:33
 * @description 实现 Feign Request 拦截器，实现 GET 传递 POJO
 */
@Component
public class FeignRequestInterceptor implements RequestInterceptor {
    private final ObjectMapper objectMapper;
    @Autowired
    public FeignRequestInterceptor(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }
    @Override
    public void apply(RequestTemplate template) {
        if ("GET".equals(template.method()) && template.body() != null) {
            try {
                JsonNode jsonNode = objectMapper.readTree(template.body());
                template.body(null);

                Map<String, Collection<String>> queries = new HashMap<>();

                // 构建 Map
                buildQuery(jsonNode, "", queries);

                // queries 就是 POJO 解析为 Map 后的数据
                template.queries(queries);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    private void buildQuery(JsonNode jsonNode, String path, Map<String, Collection<String>> queries) {
        if (!jsonNode.isContainerNode()) {
            // 如果是叶子节点
            if (jsonNode.isNull()) {
                return;
            }
            Collection<String> values = queries.get(path);
            if (CollectionUtils.isEmpty(values)) {
                values = new ArrayList<>();
                queries.put(path, values);
            }
            values.add(jsonNode.asText());
            return;
        }
        if (jsonNode.isArray()){
            // 如果是数组节点
            Iterator<JsonNode> elements = jsonNode.elements();
            while (elements.hasNext()) {
                buildQuery(elements.next(), path, queries);
            }
        } else {
            Iterator<Map.Entry<String, JsonNode>> fields = jsonNode.fields();
            while (fields.hasNext()) {
                Map.Entry<String, JsonNode> entry = fields.next();
                if (StringUtils.hasText(path)) {
                    buildQuery(entry.getValue(), path + "." + entry.getKey(), queries);
                } else {
                    // 根节点
                    buildQuery(entry.getValue(), entry.getKey(), queries);
                }
            }
        }
    }
}
```

重新启动 consumer，再次调用 `add-user`，验证结果：

![GET 成功调用远程接口](/images/spring-cloud/feign/feign-multi-params-get-user-success.png)


由此验证，GET 方式传递 POJO 成功。