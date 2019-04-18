---
title: Spring Cloud 微服务（28） --- Spring Cloud Config(二) <BR> 刷新配置
date: 2019-03-04 15:55:04
updated: 2019-03-04 15:55:04
categories: spring-cloud
tags: [SpringCloud,CloudConfig]
---

刷新配置信息的方式有三种：`手动刷新`、`半自动刷新`、`自动刷新`，其中，`半自动刷新`利用的是 `spring cloud bus`，`自动刷新`利用的是 github、gitee、gitlab 等代码托管网站的 `webhooks`

<!-- more -->

# 手动刷新

***源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-config/spring-cloud-config-refresh***

手动刷新的 config server 依然选用示例中的 `spring-cloud-config-simple-server`

## config client
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-client</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
</dependencies>
```

***bootstrap.yml***
```yml
spring:
  cloud:
    config:
      label: master
      uri: http://localhost:9090
      name: config-simple
      profile: dev
```

***application.yml***
```yml
spring:
  application:
    name: spring-cloud-config-refresh-client
server:
  port: 9091
management:
  endpoints:
    web:
      exposure:
        include: '*' # 暴露端点，用于手动刷新
  endpoint:
    health:
      show-details: always
```

***改造 config properties、config controller***
```java
@Component
@RefreshScope // 标注为配置刷新域
public class ConfigInfoProperties {

    @Value("${com.laiyy.gitee.config}")
    private String config;

    public String getConfig() {
        return config;
    }

    public void setConfig(String config) {
        this.config = config;
    }
}

@RestController
@RefreshScope // 标注为配置刷新域
public class ConfigController {

    private final ConfigInfoProperties configInfoProperties;

    @Autowired
    public ConfigController(ConfigInfoProperties configInfoProperties) {
        this.configInfoProperties = configInfoProperties;
    }

    @GetMapping(value = "/get-config-info")
    public String getConfigInfo(){
        return configInfoProperties.getConfig();
    }

}
```

## 验证

访问 http://localhost:9091/get-config-info ，观察返回值为：
```
dev 环境，git 版 spring cloud config
```

修改 https://gitee.com/laiyy0728/config-repo/blob/master/config-simple/config-simple-dev.yml 的内容为：
```yml
com:
  laiyy:
    gitee:
      config: dev 环境，git 版 spring cloud config，使用手动刷新。。。
```

再次访问 http://localhost:9091/get-config-info ，观察返回值仍为：
```
dev 环境，git 版 spring cloud config
```
这是因为没有进行手动刷新，POST 访问：http://localhost:9091/actuator/refresh ，返回信息如下：
```json
[
    "config.client.version",
    "com.laiyy.gitee.config"
]
```
控制台输出如下：
```
Fetching config from server at : http://localhost:9090
Located environment: name=config-simple, profiles=[dev], label=master, version=a04663a171b0d8f552c3d549ad38401bd6873b95, state=null
Located property source: CompositePropertySource {name='configService', propertySources=[MapPropertySource {name='configClient'}, MapPropertySource {name='https://gitee.com/laiyy0728/config-repo/config-simple/config-simple-dev.yml'}]}
```

此时，再次访问 http://localhost:9091/get-config-info ，观察返回值变为：
```
dev 环境，git 版 spring cloud config，使用手动刷新。。。
```

由此证明，手动刷新成功


---

# 半自动刷新

***源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-config/spring-cloud-config-bus***

半自动刷新依赖于 Spring Cloud Bus 总线，而 Bus 总线依赖于 RabbitMQ。 Spring Cloud Bus 刷新配置的流程图：
![Spring Cloud Bus 流程图](/images/spring-cloud/config/spring-cloud-bus.png)

***Rabbit MQ 请自行安装启动，在此不做描述***

## config server bus

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-server</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-monitor</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-bus-amqp</artifactId>
    </dependency>
</dependencies>
```

```yml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://gitee.com/laiyy0728/config-repo.git
          search-paths: config-simple
    bus:
      trace:
        enabled: true # 是否启用bus追踪
  application:
    name: spring-cloud-config-bus-server
  rabbitmq: # rabbit mq 配置
    host: 192.168.67.133
    port: 5672
    username: guest
    password: guest
server:
  port: 9090
management:
  endpoints:
    web:
      exposure:
        include: '*'
  endpoint:
    health:
      show-details: always
```

```java
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf().disable();
    }
}


@SpringBootApplication
@EnableConfigServer
public class SpringCloudConfigBusServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudConfigBusServerApplication.class, args);
    }

}
```

## config client bus

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-bus-amqp</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-client</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
</dependencies>
```

***bootstrap.yml***
```yml
spring:
  cloud:
    config:
      label: master
      uri: http://localhost:9090
      name: config-simple
      profile: test
```

***application.yml***
```yml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://gitee.com/laiyy0728/config-repo.git
          search-paths: config-simple
    bus:
      trace:
        enabled: true # \u662F\u5426\u542F\u7528bus\u8FFD\u8E2A
  application:
    name: spring-cloud-config-bus-server
  rabbitmq: # rabbit mq \u914D\u7F6E
    host: 192.168.67.133
    port: 5672
    username: guest
    password: guest
server:
  port: 9090
management:
  endpoints:
    web:
      exposure:
        include: '*'
  endpoint:
    health:
      show-details: always
```

其余 Java 类与手动刷新一致。

## 验证

访问 http://localhost:9095/get-config-info 
![Config Bus](/images/spring-cloud/config/config-bus-client.png)

将 `config-simple/config-simple-test.yml` 内容修改为
```yml
com:
  laiyy:
    gitee:
      config: test 环境，git 版 spring cloud config，bus 半自动刷新配置
```

再次访问 http://localhost:9095/get-config-info ，返回值仍为：
```
test 环境，git 版 spring cloud config
```

使用 bus 刷新配置，POST 请求 http://localhost:9095/actuator/bus-refresh ，查看控制台输出：
```
 Bean 'org.springframework.cloud.autoconfigure.ConfigurationPropertiesRebinderAutoConfiguration' of type [org.springframework.cloud.autoconfigure.ConfigurationPropertiesRebinderAutoConfiguration$$EnhancerBySpringCGLIB$$7c355e31] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
 Fetching config from server at : http://localhost:9090
 Located environment: name=config-simple, profiles=[test], label=master, version=45c17b3b2a7918ed7093251f2085641df446e961, state=null
 Located property source: CompositePropertySource {name='configService', propertySources=[MapPropertySource {name='configClient'}, MapPropertySource {name='https://gitee.com/laiyy0728/config-repo.git/config-simple/config-simple-test.yml'}]}
 No active profile set, falling back to default profiles: default
 Started application in 1.108 seconds (JVM running for 264.482)
 Received remote refresh request. Keys refreshed []
```

再次访问 http://localhost:9095/get-config-info ，返回值变为：
```
test 环境，git 版 spring cloud config，bus 半自动刷新配置
```

## refresh、bus-refresh 比较

- refresh：只能刷新单节点，即：只能刷新指定 ip 的配置信息
- bus-refresh：批量刷新，可以刷新订阅了 rabbit queue 的所有节点配置


---

# 自动刷新

自动刷新实际上很简单，只需要暴露一个 bus-refresh 节点，并在 config-server 的 git 中，配置 `webhook` 指向暴露出来的 bus-refresh 节点即可，多个 bus-refresh 节点用英文逗号分隔
![Config webhook](/images/spring-cloud/config/config-webhook.png)