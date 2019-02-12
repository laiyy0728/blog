---
title: Spring Cloud 微服务（8） --- Feign(二) <br> GZIP、配置
date: 2019-01-23 13:40:18
updated: 2019-01-23 13:40:18
categories:
    Java
tags:
    - SpringCloud
    - Feign
---


在理解了 Feign 的运行原理之后，可以很轻松的搭建起一个基于 Feign 的微服务调用。

Feign 是通过 http 调用的，那么就牵扯到一个数据大小的问题。如果不经过压缩就发送请求、获取响应，那么会因为流量过大导致浪费流量，这时就需要使用数据压缩，将大流量压缩成小流量。

<!-- more -->

# Feign GZIP 压缩

***源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-feign/spring-cloud-feign-gzip***

Spring Cloud Feign 支持对请求和响应进行 GZIP 压缩，以调高通信效率。

## 开启 gzip 压缩

application.yml

```yml
feign:
  compression:
    request:
      enabled: true
      mime-type: text/html,application/xml,application/json
      min-request-size: 2048
    response:
      enabled: true

# 开启日志
logging:
  level:
    com.laiyy.gitee.feign.springcloudfeigngzip.feign.GiteeFeignClient: debug
```

由于使用 gzip 压缩，压缩后的数据是二进制，那么在获取 Response 的时候，就不能和之前一样直接使用 String 来接收了，需要使用 ResponseEntity<byte[]> 接收

```java
@FeignClient(name = "gitee-client", url = "https://www.gitee.com/", configuration = GiteeFeignConfiguration.class)
public interface GiteeFeignClient {

    @RequestMapping(value = "/search", method = RequestMethod.GET)
    ResponseEntity<byte[]> searchRepo(@RequestParam("q") String query);

}
```

对应的 Controller 也需要改为 ResponseEntity<byte[]>
```java
@GetMapping(value = "feign-gitee")
public ResponseEntity<byte[]> feign(String query){
    return giteeFeignClient.searchRepo(query);
}
```

## 验证 gzip 压缩

开启 FeignClient 日志

### 没有使用 GZIP 压缩

在 `spring-cloud-feign-simple` 项目中，开启日志：

```yml
logging:
  level:
    com.laiyy.gitee.feign.springcloudfeignsimple.feign.GiteeFeignClient: debug
```

访问：http://localhost:8080/feign-gitee?query=spring-cloud-openfeign ，可以看到，在控制台中打印了日志信息：

![no-gzip](/images/spring-cloud/feign/no-gzip-console.png)

### 使用了 GZIP 压缩

在 `spring-cloud-feign-gzip` 中开启日志：
```yml
logging:
  level:
    com.laiyy.gitee.feign.springcloudfeigngzip.feign.GiteeFeignClient: debug
```

访问：http://localhost:8080/feign-gitee?query=spring-cloud-openfeign ，可以看到，在控制台中打印了日志信息：

![gzip](/images/spring-cloud/feign/gzip-console.png)


## 对比

### Request 对比

经过对比，可以看到在没有开启 gzip 之前，request 是：
```
---> GET https://www.gitee.com/search?q=spring-cloud-openfeign HTTP/1.1
---> END HTTP (0-byte body)
```

开启 gzip 之后，request 是：
```
---> GET https://www.gitee.com/search?q=spring-cloud-openfeign HTTP/1.1
Accept-Encoding: gzip
Accept-Encoding: deflate
---> END HTTP (0-byte body)
```

可以看到，request 中增加了 `Accept-Encoding: gzip`，证明 request 开启了 gzip 压缩。


### Response 对比

在没有开启 gzip 之前，response 是：
```
cache-control: no-cache
connection: keep-alive
content-type: text/html; charset=utf-8
date: Wed, 23 Jan 2019 07:22:45 GMT
expires: Sun, 1 Jan 2000 01:00:00 GMT
pragma: must-revalidate, no-cache, private
server: nginx
set-cookie: gitee-session-n=BAh7CEkiD3Nlc3Npb25faWQGOgZFVEkiJTIyM2VlNjhkMWVmZGJlMWY5YmIxN2M5MGVlODEzY2Q5BjsAVEkiF21vYnlsZXR0ZV9vdmVycmlkZQY7AEY6CG5pbEkiEF9jc3JmX3Rva2VuBjsARkkiMTlsSDZmQk1CWXpWWVFTSTFtbkwzb0VJTjZjbVdVKzhYZjE0ako0djIvRUk9BjsARg%3D%3D--97ef4dc9c69d79b8f6ca42b9d0b6eaeb121d8048; domain=.gitee.com; path=/; HttpOnly
set-cookie: oschina_new_user=false; path=/; expires=Sun, 23-Jan-2039 07:22:44 GMT
set-cookie: user_locale=; path=/; expires=Sun, 23-Jan-2039 07:22:44 GMT
set-cookie: aliyungf_tc=AQAAAAq0GH/W3wsAygAc2kkmdVRPGWZs; Path=/; HttpOnly
status: 200 OK
transfer-encoding: chunked
x-rack-cache: miss
x-request-id: 437df6eccbd8a2b93912a7b84644b33d
x-runtime: 0.646640
x-ua-compatible: IE=Edge,chrome=1
x-xss-protection: 1; mode=block

<!DOCTYPE html>
<html lang='zh-CN'>
<head>
<title>spring-cloud-openfeign · Search - Gitee</title>
<link href="https://assets.gitee.com/assets/favicon-e87ded4710611ed62adc859698277663.ico" rel="shortcut icon" type="image/vnd.microsoft.icon" />
<meta charset='utf-8'>
<meta content='always' name='referrer'>
<meta content='Gitee' property='og:site_name'>
...
</html>

<--- END HTTP (48623-byte body)
```

在开启 gzip 之后，response 是：
```
<--- HTTP/1.1 200 OK (987ms)
cache-control: no-cache
connection: keep-alive
content-encoding: gzip    ----------------------------- 第一处不同
content-type: text/html; charset=utf-8
date: Wed, 23 Jan 2019 07:20:59 GMT
expires: Sun, 1 Jan 2000 01:00:00 GMT
pragma: must-revalidate, no-cache, private
server: nginx
set-cookie: gitee-session-n=BAh7CEkiD3Nlc3Npb25faWQGOgZFVEkiJTVmYmMwNTQyNWU4OGMzMmYyN2M3MDQ1ZmZiNjY5ZDIzBjsAVEkiF21vYnlsZXR0ZV9vdmVycmlkZQY7AEY6CG5pbEkiEF9jc3JmX3Rva2VuBjsARkkiMVdaQ2tqYTVuTjd6WU1UKzU5R1hNbnRlbUNQaXhoSzRLRmJreXduTU51cUU9BjsARg%3D%3D--8843239d46616524d58af2611f2db9614b8518b1; domain=.gitee.com; path=/; HttpOnly
set-cookie: oschina_new_user=false; path=/; expires=Sun, 23-Jan-2039 07:20:58 GMT
set-cookie: user_locale=; path=/; expires=Sun, 23-Jan-2039 07:20:58 GMT
set-cookie: aliyungf_tc=AQAAAHojVAEggQsAygAc2ugaNNgiXCKR; Path=/; HttpOnly
status: 200 OK
transfer-encoding: chunked
x-rack-cache: miss
x-request-id: 53b45c93d5062be2c5643d9402d0a6de
x-runtime: 0.412080
x-ua-compatible: IE=Edge,chrome=1
x-xss-protection: 1; mode=block

Binary data              -------------------------------- 第二处不同
<--- END HTTP (11913-byte body)          ---------------------------- 第三处不同
```

对比可以发现：
- 在 response 的 `content-type` 上面多了一个 `content-encoding: gzip`
- 在没有开启 gzip 之前控制台打印了 html 信息，开启后没有打印，换成了 `Binary data` 二进制
- END HTTP 在没开启 gzip 之前为 48623 byte，开启后为 11913 byte

由此可以证明，response 开启 gzip 成功

---

# Feign 配置

***源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-feign/spring-cloud-feign-config***

## 对单个指定特定名称的 Feign 进行配置

在之前的例子中，在对 FeignClient 的配置中，使用的是 `@FeignClient` 的 `configuration` 属性指定的配置类，也可以使用配置文件对 `@FeignClient` 注解的接口进行配置

### FeignClient

```java
@FeignClient(name = "gitee-client", url = "https://www.gitee.com")
public interface GiteeFeignClient {

    @RequestMapping(value = "/search", method = RequestMethod.GET)
    ResponseEntity<byte[]> searchRepo(@RequestParam("q") String query);

}
```

### application.yml

```yml
feign:
  client:
    config:
      gitee-client:  # 这里指定的是 @FeignClient 的 name/value 属性的值
        connectTimeout: 5000  # 链接超时时间
        readTimeout: 5000 # 读超时
        loggerLevel: none # 日志级别
        # errorDecoder: # 错误解码器（类路径）
        # retryer: # 重试机制（类路径）
        # requestInterceptors: 拦截器配置方式 一：多个拦截器， 需要注意如果有多个拦截器，"-" 不能少
          # - Intecerptor1 类路径，
          # - Interceptpt2 类路径
        # requestInterceptors: 拦截器配置方式 二：多个拦截器，用 [Interceptor, Interceptor] 配置，需要配置类路径
        # decode404: false 是否 404 解码
        # encoder： 编码器（类路径）
        # decoder： 解码器（类路径）
        # contract： 契约（类路径）
logging:
  level:
    com.laiyy.gitee.feign.springcloudfeignconfig.feign.GiteeFeignClient: debug
```


### 验证

此时配置的 loggerLevel 为 none，不打印日志，访问： http://localhost:8080/feign-gitee?query=spring-cloud-openfeign ，可以看到控制台没有任何消息

将 loggerLevel 改为 full，再次访问可以看到打印日志消息。

将 loggerLevel 改为 feign.Logger.Level 中没有的级别，再次测试：loggerLevel: haha，可以看到控制启动报错：

```
***************************
APPLICATION FAILED TO START
***************************

Description:

Failed to bind properties under 'feign.client.config.gitee-client.logger-level' to feign.Logger$Level:

    Property: feign.client.config.gitee-client.loggerlevel
    Value: haha
    Origin: class path resource [application.yml]:7:22
    Reason: failed to convert java.lang.String to feign.Logger$Level

Action:

Update your application's configuration. The following values are valid:

    BASIC
    FULL
    HEADERS
    NONE
```

可以验证此配置是正确的。



## 对全部 FeignClient 配置

对全部 FeignClient 启用配置的方法也有两种：1、`@EnableFeignClients` 注解有一个 `defaultConfiguration` 属性，可以指定全局 FeignClient 的配置。2、使用配置文件对全局 FeignClient 进行配置

application.yml
```yml
feign:
  client:
    config:
      defautl:  # 全局的配置需要把 client-name 指定为 default
        connectTimeout: 5000  # 链接超时时间
        readTimeout: 5000 # 读超时
        loggerLevel: full # 日志级别
```

如果有多个 FeignClient，每个 FeignClient 都需要单独配置，如果有一样的配置，可以提取到全局配置中，需要注意：全局配置需要放在最后一位。




