---
title: Spring Cloud（5） --- Eureka(三) <br> 集群、Region Zone、Http Basic
date: 2019-01-19 16:47:24
updated: 2019-01-19 16:47:24
categories:
    Java
tags:
    - SpringCloud
    - Eureka
---

在了解了 Euerka 的 `REST API`、`核心类`、`核心操作`、`参数调优`等概念之后，在实际的项目中来验证这些概念。

<!-- more -->

# Eureka Server 集群扩展

在之前的例子中，和 SpringCloud 中文社区的公益 Eureka Server 都是单节点的，如果 Server 挂掉了，那么整个微服务的注册将不在可用。在这时，就需要搭建 Eureka Server 高可用集群，保证整个微服务不会因为一个 Server 挂掉而导致整个微服务不可用。

## 使用 profile，搭建高可用集群

有两种常用的方式启动多个 eureka server


### 在一个配置文件中，指定多个配置

可以使用如下配置，在一个 application.yml 文件中，配置多个 Eureka Server，相互注册指定 defaultZone，并使用 profile 区别每个 Server 的配置。

```yml
spring:
  application:
    name: spring-cloud-eureka-server-ha

---
server:
  port: 8761

eureka:
  client:
    fetch-registry: false
    register-with-eureka: false
    service-url:
      defaultZone: http://localhost:8762/eureka,http://localhost:8763/eureka
spring:
  profiles: peer1


---
spring:
  profiles: peer2
server:
  port: 8762
eureka:
  client:
    fetch-registry: false
    register-with-eureka: false
    service-url:
      defaultZone: http://localhost:8761/eureka,http://localhost:8763/eureka

---
spring:
  profiles: peer3
server:
  port: 8763
eureka:
  client:
    fetch-registry: false
    register-with-eureka: false
    service-url:
      defaultZone: http://localhost:8761/eureka,http://localhost:8762/eureka

```

### 验证多 Server 启动

使用 mvn spring-boot:run -Dspring.profiles.active=peer1
*peer1* 可以换为 peer2、peer3，启动其他的 profile

![Eureka Server Peer1](/images/spring-cloud/eureka/eureka-peer1.png)
![Eureka Server Peer2](/images/spring-cloud/eureka/eureka-peer2.png)
![Eureka Server Peer3](/images/spring-cloud/eureka/eureka-peer3.png)

在浏览器中输入： http://localhost:8761、http://localhost:8762、http://localhost:8763，都可以访问到 Eureka Server


使用这种方式启动存在的问题：
> 在一个配置文件中存在多个 Server 的配置，太过杂乱无章，不好管理
> 如果 Server 在不同的机器上，由于 ip 地址不同，在第一个 Server 启动时由于找不到注册中心，必报错，当第二个 Server 启动后正常


### 使用多配置文件

将 application.yml 复制多份，改名为 `application-peer1.yml`、`application-peer2.yml`、`application-peer3.yml`，然后使用 mvn spring-boot:run -Dspring.profiles.active=peer1 启动项目。

为验证此种配置方式可用，将 peer3 的端口该为 8764，启动测试

![Eureka Server Peer4](/images/spring-cloud/eureka/eureka-peer4.png)

使用这种方式的问题：
> 可能存在同样的配置在多个配置文件都存在，需要修改时需要修改每个文件，太过冗余
> Server 在不同机器上的时候，出现的问题和 [在一个配置文件中，指定多个配置](#在一个配置文件中，指定多个配置) 的问题一致

### 接口验证

```java
@SpringBootApplication
@EnableEurekaServer
@RestController
public class SpringCloudEurekaServerHaApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudEurekaServerHaApplication.class, args);
    }


    @Autowired
    private EurekaClientConfigBean eurekaClientConfigBean;

    @GetMapping(value = "eureka-service-url")
    public Object getEurekaServerUrl() {
        return eurekaClientConfigBean.getServiceUrl();
    }

}

```

重启 3 个Server，浏览器访问 http://localhost:8764/eureka-service-url

![Service Urls](/images/spring-cloud/eureka/service-url.png)

可以看到，peer3 中注册了两个 eureka server。

### Eureka Client 注册到多 Server

Client 注册到多 Server，只需要在配置文件中指定对应 Server 的 defauleZone 即可。

```yml
spring:
  application:
    name: eureka-client 
server:
  port: 8001 

eureka:
  instance:
    hostname: localhost 
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/,http://localhost:8762/eureka/,http://localhost:8764/eureka/,
  server:
    enable-self-preservation: false 
```

---

# 使用 Region、Zone 搭建高可用集群


配置文件 application-zone1a.yml：
```yml
spring:
  application:
    name: spring-cloud-eureka-server-region-zone
server:
  port: 8761
eureka:
  client:
    fetch-registry: false
    register-with-eureka: false
    service-url:
      zone1: http://localhost:8761/eureka/,http://localhost:8762/eureka/
      zone2: http://localhost:8763/eureka/,http://localhost:8764/eureka/
    region: region-east # 设置 region
    availability-zones:
      region-east: zone1,zone2 # 设置可用 region-zone
  instance:
    hostname: localhost
    prefer-ip-address: true
    metadata-map:
      zone: zone1 # 设置 zone
```

application-zone1b.yml： 将 server.port 修改为 8762
application-zone2a.yml： 将 server.port 修改为 8763，eureka.instance.metadata-map.zone 修改为 zone2
application-zone2b.yml： 将 server.port 修改为 8764，eureka.instance.metadata-map.zone 修改为 zone2

## 验证 Eureka Server

在浏览器访问 http://localhost:8761

![Region Zone Server](/images/spring-cloud/eureka/region-zone-server.png)

## 创建 Eureka Client

创建两个 Eureka Client，分别对应两个 Zone

### application-zone1.yml
```yml
server:
  port: 8081
spring:
  application:
    name: spring-cloud-eureka-client-region-zone
eureka:
  instance:
    metadata-map:
      zone: zone1
  client:
    region: region-east
    availability-zones:
      region-east: zone1,zone2
    service-url:
      zone1: http://localhost:8761/eureka/,http://localhost:8762/eureka/
      zone2: http://localhost:8763/eureka/,http://localhost:8764/eureka/
```

application-zone2.yml：修改 server.port=8082，eureka.instance.metadata-map.zone=zone2

暴露服务端点
application.yml：
```yml
management:
  endpoints:
    web:
      exposure:
        include: '*'
```

### 引入 pom 依赖：
```xml
<dependencies>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>

<!-- 暴露端点 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
</dependencies>
```

### 验证 region、zone

访问 client 暴露的环境端点，验证 region、zone

http://localhost:8081/actuator/env、http://localhost:8082/actuator/env

![Client Zone1](/images/spring-cloud/eureka/client-zone1.png)
![Client Zone2](/images/spring-cloud/eureka/client-zone2.png)

由此，可以验证 client1、client2 的 zone 是指定的 zone。

---

# 开启 Http Basic

现在的实例中，访问 Eureka Server 是不需要用户名、密码的，不需要安全验证。为了防止微服务暴露，可以开启 Http Basic 安全教研。

## Eureka Server 开启 Http Basic

### 引入 pom 依赖

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
</dependencies>
```

### 创建配置文件

```yml
server:
  port: 8761
spring:
  security:
    user:
      name: laiyy  # 访问 Eureka Server 的用户名
      password: 123456 # 访问 Eureka Server 的密码

eureka:
  client:
    service-url:
      defaultZone: http://localhost:${server.port:8761}/eureka/
    register-with-eureka: false
    fetch-registry: false
```

### 访问 http://localhost:8761

![HTTP Basic](/images/spring-cloud/eureka/http-basic.png)


## Eureka Client 开启 Http Basic

### 引入 pom 依赖

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
    name: spring-cloud-eureka-client-http-basic

eureka:
  client:
    security:
      basic:
        user: laiyy
        password: 123456
    service-url:
      defaultZone: http://${eureka.client.security.basic.user}:${eureka.client.security.basic.password}@localhost:8761/eureka
  instance:
    prefer-ip-address: true
    instance-id: ${spring.application.name}:${server.port}

server:
  port: 8081
```

需要注意，defaultZone 需要设置为： http://user:password@ip:port/eureka/

### 启动 Eureka Client，验证 Http Basic

在启动 Client 后，观察日志，可以看到出现了 403 错误：
![Eureka Client Http Basic Error](/images/spring-cloud/eureka/eureka-client-http-basic-error.png)


明明已经指定了 Eureka Server 的用户名、密码、ip、端口，为什么还是注册失败？
是因为 Http Basic 默认是同源的，而 client、server 的 ip、端口不一致，会出现跨域访问请求，导致 403.

解决办法：在 Eureka Server 端关闭 csrf 访问。

```java
@EnableWebSecurity
public class HttpBasicConfiguration extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        super.configure(http);
        http.csrf().disable();
    }
}
```

重新启动 Server、Client，访问 Server，可以看到 Client 注册成功

![Eureka Client Http Basic](/images/spring-cloud/eureka/client-http-basic.png)