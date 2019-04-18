---
title: Spring Cloud 微服务（6） --- Eureka(四) <br> Https
date: 2019-01-20 14:45:20
updated: 2019-01-20 14:45:20
categories:
    spring-cloud
tags:
    - SpringCloud
    - Eureka
---

在生产环境下，一般来说都是 https 协议访问，现在的 http 协议访问可能会出现问题，在 Eureka Server、Client 中开启 Https 访问。

<!-- more -->


HTTP Basic 基于 base64 编码，容易被抓包，如果暴露在公网会非常不安全，可以通过开启 https 达到保护数据的目的。

## Server 证书生成

```
keytool -genkeypair -alias server -storetype PKCS12 -keyalg RSA -keysize 2048 -keystore server.p12 -validity 3650
```

密码为：123456

![GenKey Server](/images/spring-cloud/eureka/key-gen-server.png)

在当前目录生成了一个 server.p12 文件

## Client 证书生成

```
keytool -genkeypair -alias client -storetype PKCS12 -keyalg RSA -keysize 2048 -keystore client.p12 -validity 3650
```

密码为：654321

![GenKey Client](/images/spring-cloud/eureka/key-gen-client.png)

在当前目录生成了一个 client.p12 文件

![P12 文件](/images/spring-cloud/eureka/p12.png)

## 导出 p12 文件

```
keytool -export -alias server -file server.crt --keystore server.p12
keytool -export -alias client -file client.crt --keystore client.p12
```

![Export p12](/images/spring-cloud/eureka/export-p12.png)
![crt file](/images/spring-cloud/eureka/crt.png)


## 信任证书

### Client 信任 Server 证书

将 server.crt 导入 client.p12

```
keytool -import -alias server -file server.crt -keystore client.p12
```

秘钥口令是 client.p12 的口令

![client 信任 server](/images/spring-cloud/eureka/server-crt-to-client-p12.png)

### Server 信任 Client 证书

将 client.crt 导入 server.p12

```
keytool -import -alias client -file client.crt -keystore server.p12
```

秘钥口令是 server.p12 的口令

![Server 信任 Client](/images/spring-cloud/eureka/client-crt-to-server-p12.png)

## Eureka Server

***源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-eureka/spring-cloud-eureka-server-https***

将生成的最后的 server.p12 文件放在 resources 下

### application.yml
```yml
server:
  port: 8761
  ssl:
    enabled: true
    key-store-type: PKCS12 # type 与 keytool 的 storetype 一致
    key-alias: server # 与 keytool 的 alias 一致
    key-store: classpath:server.p12 # p12 文件地址
    key-store-password: 123456 # server.p12 口令

eureka:
  instance:
    hostname: localhost
    secure-port: ${server.port} # https 端口
    secure-port-enabled: true # 是否开启 https port
    non-secure-port-enabled: false 
    home-page-url: https://${eureka.instance.hostname}:${server.port} # https 协议
    status-page-url: https://${eureka.instance.hostname}:${server.port} # https 协议
  client:
    register-with-eureka: false
    fetch-registry: false
    service-url:
      defaultZone: https://${eureka.instance.hostname}:${server.port}/eureka/ # https 协议
```

### 验证 Eureka Server

访问 http://localhost:8761

![Http 协议](/images/spring-cloud/eureka/http-server.png)

访问 https://localhost:8761

![Https 协议](/images/spring-cloud/eureka/https-server.png)

## Eureka Client

***源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-eureka/spring-cloud-eureka-client-https***

Client 只在连接 Eureka Server 的时候使用 https 协议，如果要全局都使用 https，则和 Server 的 https 配置一致，只需要将配置换成 client.p12 的配置即可。

### application.yml

```yml
server:
  port: 8081

spring:
  application:
    name: client1

eureka:
  client:
    securePortEnabled: true
    ssl:
      key-store: client.p12
      key-store-password: 654321
    serviceUrl:
      defaultZone: https://localhost:8761/eureka/
```

### Https 连接配置

```java
@Configuration
public class EurekaHttpsClientConfiguration {

    @Value("${eureka.client.ssl.key-store}")
    private String ketStoreFileName;

    @Value("${eureka.client.ssl.key-store-password}")
    private String ketStorePassword;

    @Bean
    public DiscoveryClient.DiscoveryClientOptionalArgs discoveryClientOptionalArgs() throws CertificateException, NoSuchAlgorithmException, KeyStoreException, IOException, KeyManagementException {
        EurekaJerseyClientImpl.EurekaJerseyClientBuilder builder = new EurekaJerseyClientImpl.EurekaJerseyClientBuilder();
        builder.withClientName("eureka-https-client");
        URL url = this.getClass().getClassLoader().getResource(ketStoreFileName);
        SSLContext sslContext = new SSLContextBuilder()
                .loadTrustMaterial(url, ketStorePassword.toCharArray()).build();
        builder.withCustomSSL(sslContext);
        builder.withMaxTotalConnections(10);
        builder.withMaxConnectionsPerHost(10);

        DiscoveryClient.DiscoveryClientOptionalArgs optionalArgs = new DiscoveryClient.DiscoveryClientOptionalArgs();
        optionalArgs.setEurekaJerseyClient(builder.build());

        return optionalArgs;
    }

}
```

### 验证 Client

访问 https://localhost:8761

![验证 client](/images/spring-cloud/eureka/check-client-connect-server.png)

### 使用 http 注册

![Client 使用 Http 访问 Server](/images/spring-cloud/eureka/client-http-connect-server.png)

