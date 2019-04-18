---
title: Spring Cloud 微服务 （10） --- Feign(四)  <BR> 文件上传、首次调用失败问题
date: 2019-01-24 17:05:01
updated: 2019-01-24 17:05:01
categories:
    spring-cloud
tags:
    - SpringCloud
    - Feign
---


Feign 在远程调用时，除了 GET 方式传递 POJO 外，还有几个很重要的功能：`文件上传`、`调用返回图片流`、`传递 Token` 等

<!-- more -->

# 文件上传

***源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-feign/spring-cloud-feign-file***

Feign 的子项目 feign-form(https://github.com/OpenFeign/feign-form) 支持文件上传，其中实现了上传所需要的 Encoder

模拟文件上传：`spring-cloud-feign-file-server`、`spring-cloud-feign-file-client`，其中 server 模拟文件服务器，作为服务提供者；client 模拟文件上传，通过 FeignClient 发送文件到文件服务器

## FileClient

### pom 依赖

```xml
<!-- eureka client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>

<!-- feign -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>

<!-- Feign文件上传依赖-->
<dependency>
    <groupId>io.github.openfeign.form</groupId>
    <artifactId>feign-form</artifactId>
    <version>3.0.3</version>
</dependency>
<dependency>
    <groupId>io.github.openfeign.form</groupId>
    <artifactId>feign-form-spring</artifactId>
    <version>3.0.3</version>
</dependency>
```

### 配置文件

```yml
spring:
  application:
    name: spring-cloud-feign-file-client

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    instance-id: ${spring.application.name}:${server.port}
    prefer-ip-address: true
server:
  port: 8888
```

### 启动类、FeignClient、配置、Controller
```java
// 启动类
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients
public class SpringCloudFeignFileClientApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringCloudFeignFileClientApplication.class, args);
    }
}

// FeignClient
@FeignClient(value = "spring-cloud-feign-file-server", configuration = FeignMultipartConfiguration.class)
public interface FileUploadFeignClient {

    /**
     * feign 上传图片
     *
     * produces、consumes 必填
     * 不要将 @RequestPart 写成 @RequestParam
     *
     * @param file 上传的文件
     * @return 上传的文件名
     */
    @RequestMapping(value = "/upload-file", method = RequestMethod.POST, produces = MediaType.APPLICATION_JSON_UTF8_VALUE, consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    String fileUpload(@RequestPart(value = "file")MultipartFile file);

}

// configuration
@Configuration
public class FeignMultipartConfiguration {

    /**
     * Feign Spring 表单编码器
     * @return 表单编码器
     */
    @Bean
    @Primary
    @Scope("prototype")
    public Encoder multipartEncoder(){
        return new SpringFormEncoder();
    }

}

// Controller
@RestController
public class FileUploadController {

    private final FileUploadFeignClient feignClient;

    @Autowired
    public FileUploadController(FileUploadFeignClient feignClient) {
        this.feignClient = feignClient;
    }

    @PostMapping(value = "upload")
    public String upload(MultipartFile file){
        return feignClient.fileUpload(file);
    }

}
```

## FileServer

### pom

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

### 配置文件

```yml
spring:
  application:
    name: spring-cloud-feign-file-server


eureka:
  instance:
    prefer-ip-address: true
    instance-id: ${spring.application.name}:${server.port}
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
server:
  port: 8889
```

### 启动类、Controller

```java
@SpringBootApplication
@EnableDiscoveryClient
public class SpringCloudFeignFileServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudFeignFileServerApplication.class, args);
    }

}

// Controller 模拟文件上传的处理
@RestController
public class FileUploadController {

    @PostMapping(value = "/upload-file")
    public String fileUpload(MultipartFile file) {
        return file.getOriginalFilename();
    }

}
```

## 验证文件上传

POST MAN 调用 client 上传接口

![file upload](/images/spring-cloud/feign/file-upload.png)

---

# 图片流

***源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-feign/spring-cloud-feign-file***

通过 Feign 返回图片，一般是字节数组

在`文件上传`代码的基础上，再加上图片获取

## FeignClient

```java
/**
  * 获取图片
  * @return 图片
  */
@RequestMapping(value = "/get-img")
ResponseEntity<byte[]> getImage();


@GetMapping(value = "/get-img")
public ResponseEntity<byte[]> getImage(){
    return feignClient.getImage();
}
```

## FeignServer
```java
@GetMapping(value = "/get-img")
public ResponseEntity<byte[]> getImages() throws IOException {
    FileSystemResource resource = new FileSystemResource(getClass().getResource("/").getPath() + "Spring-Cloud.png");
    HttpHeaders headers = new HttpHeaders();
    headers.add("Content-Type", MediaType.APPLICATION_OCTET_STREAM_VALUE);
    headers.add("Content-Disposition", "attachment; filename=Spring-Cloud.png");
    return  ResponseEntity.status(HttpStatus.OK).headers(headers).body(FileCopyUtils.copyToByteArray(resource.getInputStream()));
}
```

## 验证

在浏览器访问：http://localhost:8888/get-img ，实现图片流下载

---

# Feign 传递 Headers

***源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-feign/spring-cloud-feign-multi-params***

在认证、鉴权中，无论是哪种权限控制框架，都需要传递 header，但在使用 Feign 的时候，会发现外部请求 ServiceA 时，可以获取到 header，但是在 ServiceA 调用 ServiceB 时，ServiceB 无法获取到 Header，导致 Header 丢失。

在 `spring-cloud-feign-multi-params` 基础上，实现传递 Header。

## 验证 Header 无法传递问题

![post man 传递 header1](/images/spring-cloud/feign/post-man-header1.png)

consumer 打印 header
![Consumer 打印 header](/images/spring-cloud/feign/consumer-header.png)

provider 打印 header
![Provider 打印 header](/images/spring-cloud/feign/provider-header.png)

## HeaderInterceptor

在 Consumer 增加 HeaderInterceptor，做 header 传递

```java
@Component
public class FeignHeaderInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate template) {
        if (null == getRequest()){
            return;
        }
        template.header("oauth-token", getHeaders(getRequest()).get("oauth-token"));
    }

    private HttpServletRequest getRequest(){
        try {
            return ((ServletRequestAttributes) RequestContextHolder.currentRequestAttributes()).getRequest();
        }catch (Exception e){
            return null;
        }
    }

    private Map<String, String> getHeaders(HttpServletRequest request) {
        Map<String, String> headers = new HashMap<>();
        Enumeration<String> headerNames = request.getHeaderNames();
        while (headerNames.hasMoreElements()) {
            String key = headerNames.nextElement();
            String value = request.getHeader(key);
            headers.put(key, value);
        }
        return headers;
    }

}
```

## 验证 header

用 postman 重新请求一遍，查看 provider 控制台打印：

![provider header](/images/spring-cloud/feign/provider-header-success.png)