---
title: Spring Cloud 微服务（28） --- Spring Cloud Config(二) <BR> 刷新配置
date: 2019-03-04 15:55:04
updated: 2019-03-04 15:55:04
categories: Java
tags: [SpringCloud,CloudConfig]
---

刷新配置信息的方式有三种：`手动刷新`、`半自动刷新`、`自动刷新`，其中，`半自动刷新`利用的是 `spring cloud bus`，`自动刷新`利用的是 github、gitee、gitlab 等代码托管网站的 `webhooks`

<!-- more -->

# 手动刷新

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

半自动刷新依赖于 Spring Cloud Bus 总线，而 Bus 总线依赖于 RabbitMQ。 Spring Cloud Bus 刷新配置的流程图：
![Spring Cloud Bus 流程图](/images/spring-cloud/config/spring-cloud-bus.png)

***Rabbit MQ 请自行安装启动，在此不做描述***

## config server bus