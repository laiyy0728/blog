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

