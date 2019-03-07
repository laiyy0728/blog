---
title: Spring Cloud 微服务（30） --- Spring Cloud Config(四) <BR>  Spring Cloud 配置、高可用
date: 2019-03-06 11:31:06
updated: 2019-03-06 11:31:06
categories:
    Java
tags: [SpringCloud, CloudConfig]
---


# 本地参数覆盖远程参数

```yml
spring:
  cloud:
    config:
      allow-override: true
      override-none: true
      override-system-properties: false
```

<!-- more -->

- allow-override：标识 `override-system-properties` 是否启用，默认为 true，设置为 false 时，意味着禁用用户的设置
- override-none：当此项为 true，`override-override` 为 true，外部的配置优先级更低，而且不能覆盖任何存在的属性源。默认为 false
- override-system-properties：用来标识外部配置是否能够覆盖系统配置，默认为 true

```java
@ConfigurationProperties("spring.cloud.config")
public class PropertySourceBootstrapProperties {
    private boolean overrideSystemProperties = true;
    private boolean allowOverride = true;
    private boolean overrideNone = false;

    public PropertySourceBootstrapProperties() {
    }

    // 省略 getter、setter
}
```


---

# 客户端功能扩展

## 客户端自动刷新

***源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-config/spring-cloud-autoconfig***

在有些应用上，不需要再服务端批量推送的时候，客户端本身需要获取变化参数的情况下，使用客户端的自动刷新能完成此功能。

***config server*** 依然采用 `spring-cloud-config-simple-server`，基础配置不变，配置文件 repo 依然是 https://gitee.com/laiyy0728/config-repo

### 配置拉取、刷新二方库

新建一个二方库（spring-cloud-autoconfig-refresh），用于其他项目引入，以自动刷新配置（用于多个子项目使用同一个配置中心，自动刷新）

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-client</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-autoconfigure</artifactId>
    </dependency>
</dependencies>
```

整个二方库只有这一个类，作用是获取定时刷新时间，并刷新配置
```java
@Configuration
@ConditionalOnClass(RefreshEndpoint.class)
@ConditionalOnProperty("spring.cloud.config.refreshInterval")
@AutoConfigureAfter(RefreshAutoConfiguration.class)
@EnableScheduling
public class SpringCloudAutoconfigRefreshApplication implements SchedulingConfigurer {

    private static final Logger LOGGER = LoggerFactory.getLogger(SpringCloudAutoconfigRefreshApplication.class);

    @Autowired
    public SpringCloudAutoconfigRefreshApplication(RefreshEndpoint refreshEndpoint) {
        this.refreshEndpoint = refreshEndpoint;
    }

    @Value("${spring.cloud.config.refreshInterval}")
    private long refreshInterval;

    private final RefreshEndpoint refreshEndpoint;


    @Override
    public void configureTasks(ScheduledTaskRegistrar scheduledTaskRegistrar) {
        final long interval = getRefreshIntervalilliseconds();
        LOGGER.info(">>>>>>>>>>>>>>>>>>>>>> 定时刷新延迟 {} 秒启动，每 {} 毫秒刷新一次配置 <<<<<<<<<<<<<<<<", refreshInterval, interval);
        scheduledTaskRegistrar.addFixedDelayTask(new IntervalTask(refreshEndpoint::refresh, interval, interval));
    }

    /**
     * 返回毫秒级时间间隔
     */
    private long getRefreshIntervalilliseconds() {
        return refreshInterval * 1000;
    }

}
```

***/resources/META-INF/spring.factories***
```java
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.laiyy.gitee.config.springcloudautoconfigrefresh.SpringCloudAutoconfigRefreshApplication
```

### 客户端引入二方库

创建客户端项目(spring-cloud-autoconfig-client)

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-client</artifactId>
    </dependency>

    <!-- 将配置好的自刷刷新作为二方库引入 -->
    <dependency>
        <groupId>com.laiyy.gitee.config</groupId>
        <artifactId>spring-cloud-autoconfig-refresh</artifactId>
        <version>0.0.1-SNAPSHOT</version>
    </dependency>
</dependencies>
```

***bootstrap.yml***
```yml
spring:
  cloud:
    config:
      uri: http://localhost:9090
      label: master
      name: config-simple
      profile: dev
```

***application.yml***
```yml
server:
  port: 9091
spring:
  application:
    name: spring-cloud-autoconfig-client
  cloud:
    config:
      refreshInterval: 10 # 延迟时间、定时刷新时间
```

其余配置与 `spring-cloud-config-simple-client` 一致


### 验证

启动项目，访问 http://localhost:9090/get-config-info ，正常返回信息。修改 config repo 配置文件，等待 10 秒后，再次访问，可见返回信息已经变为修改后信息。
查看 client 控制台，可见定时刷新日志
```
Exposing 2 endpoint(s) beneath base path '/actuator'
>>>>>>>>>>>>>>>>>>>>>> 定时刷新延迟 10 秒启动
No TaskScheduler/ScheduledExecutorService bean found for scheduled processing
Tomcat started on port(s): 9091 (http) with context path ''
Started SpringCloudAutoconfigClientApplication in 4.361 seconds (JVM running for 5.089)
Initializing Spring DispatcherServlet 'dispatcherServlet'
Initializing Servlet 'dispatcherServlet'
Fetching config from server at : http://localhost:9090      ------------------ 第一次请求
Completed initialization in 7 ms
Located environment: name=config-simple, profiles=[dev], label=master, version=00324826262afd5178a648a469247f4fffea945e, state=null
Bean 'org.springframework.cloud.autoconfigure.ConfigurationPropertiesRebinderAutoConfiguration' of type [org.springframework.cloud.autoconfigure.ConfigurationPropertiesRebinderAutoConfiguration$$EnhancerBySpringCGLIB$$f67277ed] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
Fetching config from server at : http://localhost:9090       ------------------ 定时刷新配置
Located environment: name=config-simple, profiles=[dev], label=master, version=00324826262afd5178a648a469247f4fffea945e, state=null
Located property source: CompositePropertySource {name='configService', propertySources=[MapPropertySource {name='configClient'}, MapPropertySource {name='https://gitee.com/laiyy0728/config-repo/config-simple/config-simple-dev.yml'}]}

... 省略其他多次刷新
```

## 客户端回退

客户端回退机制，可以在出现网络中断时、或者配置服务因维护而关闭时，使得客户端可以正常使用。当启动回退时，客户端适配器将配置“缓存”到计算机中。要启用回退功能，只需要指定缓存存储的位置即可。


***源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-config/spring-cloud-config-fallback***


***config server*** 依然采用 `spring-cloud-config-simple-server`，基础配置不变，配置文件 repo 依然是 https://gitee.com/laiyy0728/config-repo

### 二方库

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-config</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.security</groupId>
        <artifactId>spring-security-rsa</artifactId>
        <version>1.0.7.RELEASE</version>
    </dependency>
</dependencies>
```

***config-client.properties***：用于设置是否开启配置
```
spring.cloud.config.enabled=false
```

***/resources/META-INF/spring.factories***
```java
org.springframework.cloud.bootstrap.BootstrapConfiguration=\
  com.laiyy.gitee.config.springcloudconfigfallbackautorefresh.ConfigServerBootStrap
```

```java
// 用于拉取远程配置文件，并保存到本地
@Order(0)
public class FallbackableConfigServerPropertySourceLocator extends ConfigServicePropertySourceLocator {

    private static final Logger LOGGER = LoggerFactory.getLogger(FallbackableConfigServerPropertySourceLocator.class);

    private boolean fallbackEnabled;

    private String fallbackLocation;

    @Autowired(required = false)
    private TextEncryptor textEncryptor;

    public FallbackableConfigServerPropertySourceLocator(ConfigClientProperties defaultProperties, String fallbackLocation) {
        super(defaultProperties);
        this.fallbackLocation = fallbackLocation;
        this.fallbackEnabled = !StringUtils.isEmpty(fallbackLocation);
    }

    @Override
    public PropertySource<?> locate(Environment environment){
        PropertySource<?> propertySource = super.locate(environment);
        if (fallbackEnabled && propertySource != null){
            storeLocally(propertySource);
        }
        return propertySource;
    }

    /**
     * 转换配置文件
     */
    private void storeLocally(PropertySource propertySource){
        StringBuilder builder = new StringBuilder();
        CompositePropertySource source = (CompositePropertySource) propertySource;
        for (String propertyName : source.getPropertyNames()) {
            Object property = source.getProperty(propertyName);
            if (textEncryptor != null){
                property = "{cipher}" + textEncryptor.encrypt(String.valueOf(property));
            }
            builder.append(propertyName).append("=").append(property).append("\n");
        }
        LOGGER.info(">>>>>>>>>>>>>>>>> file content: {} <<<<<<<<<<<<<<<<<<<", builder);
        saveFile(builder.toString());
    }

    /**
     * 保存配置到本地
     * @param content 配置内容
     */
    private void saveFile(String content){
        File file = new File(fallbackLocation + File.separator + ConfigServerBootStrap.FALLBACK_NAME);
        try {
            FileCopyUtils.copy(content.getBytes(), file);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}


// 用于判断从远程拉取配置文件，还是从本地拉取（spring boot 2.0，spring cloud F版）
@Configuration
@EnableConfigurationProperties
@PropertySource(value = {"config-client.properties","file:{spring.cloud.config.fallback-location:}/fallback.properties"}, ignoreResourceNotFound = true)
public class ConfigServerBootStrap {

    public static final String FALLBACK_NAME = "fallback.properties";

    private final ConfigurableEnvironment configurableEnvironment;

    @Autowired
    public ConfigServerBootStrap(ConfigurableEnvironment configurableEnvironment) {
        this.configurableEnvironment = configurableEnvironment;
    }

    @Value("${spring.cloud.config.fallback-location:}")
    private String fallbackLocation;

    @Bean
    public ConfigClientProperties configClientProperties(){
        ConfigClientProperties configClientProperties = new ConfigClientProperties(this.configurableEnvironment);
        configClientProperties.setEnabled(false);
        return configClientProperties;
    }

    @Bean
    public FallbackableConfigServerPropertySourceLocator fallbackableConfigServerPropertySourceLocator(){
        ConfigClientProperties client = configClientProperties();
        return new FallbackableConfigServerPropertySourceLocator(client, fallbackLocation);
    }

}
```

在 SpringBoot 1.0、Spring Cloud G 版中，会启动报错：
```
***************************
APPLICATION FAILED TO START
***************************

Description:

The bean 'configClientProperties', defined in class path resource [com/laiyy/gitee/config/springcloudconfigfallbackautorefresh/ConfigServerBootStrap.class], could not be registered. A bean with that name has already been defined in class path resource [org/springframework/cloud/config/client/ConfigServiceBootstrapConfiguration.class] and overriding is disabled.

Action:

Consider renaming one of the beans or enabling overriding by setting spring.main.allow-bean-definition-overriding=true

2019-03-07 10:10:11.230 ERROR 13828 --- [           main] o.s.boot.SpringApplication               : Application run failed

org.springframework.beans.factory.support.BeanDefinitionOverrideException: Invalid bean definition with name 'configClientProperties' defined in class path resource [com/laiyy/gitee/config/springcloudconfigfallbackautorefresh/ConfigServerBootStrap.class]: Cannot register bean definition [Root bean: class [null]; scope=; abstract=false; lazyInit=false; autowireMode=3; dependencyCheck=0; autowireCandidate=true; primary=false; factoryBeanName=com.laiyy.gitee.config.springcloudconfigfallbackautorefresh.ConfigServerBootStrap; factoryMethodName=configClientProperties; initMethodName=null; destroyMethodName=(inferred); defined in class path resource [com/laiyy/gitee/config/springcloudconfigfallbackautorefresh/ConfigServerBootStrap.class]] for bean 'configClientProperties': There is already [Root bean: class [null]; scope=; abstract=false; lazyInit=false; autowireMode=3; dependencyCheck=0; autowireCandidate=true; primary=false; factoryBeanName=org.springframework.cloud.config.client.ConfigServiceBootstrapConfiguration; factoryMethodName=configClientProperties; initMethodName=null; destroyMethodName=(inferred); defined in class path resource [org/springframework/cloud/config/client/ConfigServiceBootstrapConfiguration.class]] bound.
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.registerBeanDefinition(DefaultListableBeanFactory.java:894) ~[spring-beans-5.1.2.RELEASE.jar:5.1.2.RELEASE]
	at org.springframework.context.annotation.ConfigurationClassBeanDefinitionReader.loadBeanDefinitionsForBeanMethod(ConfigurationClassBeanDefinitionReader.java:274) ~[spring-context-5.1.2.RELEASE.jar:5.1.2.RELEASE]
    ....
```

如果按照报错提示，增加了 `spring.main.allow-bean-definition-overriding=true` 的配置，没有任何作用；如果修改了 bean 名称
```java
@Bean(name="clientProperties")
public ConfigClientProperties configClientProperties(){
    ConfigClientProperties configClientProperties = new ConfigClientProperties(this.configurableEnvironment);
    configClientProperties.setEnabled(false);
    return configClientProperties;
}
```
会有如下报错：
```java
***************************
APPLICATION FAILED TO START
***************************

Description:

Method configClientProperties in org.springframework.cloud.config.client.ConfigClientAutoConfiguration required a single bean, but 2 were found:
	- configClientProperties: defined by method 'configClientProperties' in class path resource [org/springframework/cloud/config/client/ConfigClientAutoConfiguration.class]
	- clientProperties: defined by method 'configClientProperties' in class path resource [com/laiyy/gitee/config/springcloudconfigfallbackautorefresh/ConfigServerBootStrap.class]


Action:

Consider marking one of the beans as @Primary, updating the consumer to accept multiple beans, or using @Qualifier to identify the bean that should be consumed
```


原因：`ConfigClientProperties` 在初始化时已经默认单例加载。即：这个 bean 不能被重新注册到 spring 容器中。
解决办法：将 spring 容器已经加载的单例的 `ConfigClientProperties` 注入进来，并在构造中设置为 false 即可
```java
@Configuration
@EnableConfigurationProperties
@PropertySource(value = {"configClient.properties", "file:${spring.cloud.config.fallbackLocation:}/fallback.properties"}, ignoreResourceNotFound = true)
public class ConfigServerBootStrap {

    public static final String FALLBACK_NAME = "fallback.properties";

    private final ConfigurableEnvironment configurableEnvironment;

    private final ConfigClientProperties configClientProperties;

    @Autowired
    public ConfigServerBootStrap(ConfigurableEnvironment configurableEnvironment, ConfigClientProperties configClientProperties) {
        this.configurableEnvironment = configurableEnvironment;
        this.configClientProperties = configClientProperties;

        this.configClientProperties.setEnabled(false);
    }

    @Value("${spring.cloud.config.fallbackLocation:}")
    private String fallbackLocation;

    @Bean
    public FallbackableConfigServerPropertySourceLocator fallbackableConfigServerPropertySourceLocator() {
        return new FallbackableConfigServerPropertySourceLocator(configClientProperties, fallbackLocation);
    }
}
```

### config client

***bootstrap.yml***
```yml
spring:
  cloud:
    config:
      uri: http://localhost:9090
      label: master
      name: config-simple
      profile: dev
      fallbackLocation: E:\\springcloud
```

***application.yml***
```yml
server:
  port: 9091
spring:
  application:
    name: spring-cloud-autoconfig-client
  main:
    allow-bean-definition-overriding: true
management:
  endpoints:
    web:
      exposure:
        include: '*'
  endpoint:
    health:
      show-details: always
```

其余配置、JAVA 类不变

### 验证

启动 config client，查看控制台，可见打印了 2 次远程拉取同步本地文件的信息：
```
>>>>>>>>>>>>>>>>> file content: config.client.version=ee39bf20c492b27c2d1b1d0ff378ad721e79a758
com.laiyy.gitee.config=dev 环境，git 版 spring cloud config-----!
 <<<<<<<<<<<<<<<<<<<
```

查看本地 E:\\springcloud 文件夹，可见多了一个 `fallback.properties` 文件
![fallback local properties](/images/spring-cloud/config/fallback-local-properties.png)

文件内容：
![fallback local properties content](/images/spring-cloud/config/fallback-local-properties-content.png)

更新 config repo 的对应配置文件后，POST 访问 config client 刷新端口：http://localhost:9091/actuator/refresh 可见控制台再次打印同步本地文件信息。此时停止 config server 访问，再次访问 http://localhost:9091/get-config-info ，返回的信息是同步后的更新结果，由此验证客户端回退成功。