---
title: Spring Cloud 微服务（31） --- Spring Cloud Config(五) <BR> 高可用
date: 2019-03-07 10:29:50
updated: 2019-03-07 10:29:50
categories:
    Java
tags:
    [SpringCloud, CloudConfig]
---

对于线上的生产环境，通常对其都是有很高的要求，其中，高可用是不可或缺的一部分。必须保证服务是可用的，才能保证系统更好的运行，这是业务稳定的保证。
高可用一般分为两种：`客户端高可用`、`服务端高可用`

<!-- more -->

# 客户端高可用

***源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-config/spring-cloud-config-ha/spring-cloud-config-ha-client***

`客户端高可用` 主要解决当前服务端不可用哪个的情况下，客户端依然可用正常启动。从客户端触发，不是增加配置中心的高可用性，而是降低客户端对配置中心的依赖程度，从而提高整个分布式架构的健壮性。

## 实现

### 配置的自动装配

**pom.xml**
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-client</artifactId>
    </dependency>
</dependencies>
```

**配置文件解析**
```java
@Component
@ConfigurationProperties(prefix = ConfigSupportProperties.CONFIG_PREFIX)
public class ConfigSupportProperties {

    /**
     * 加载的配置文件前缀
     */
    public static final String CONFIG_PREFIX = "spring.cloud.config.backup";

    /**
     * 默认文件名
     */
    private final String DEFAULT_FILE_NAME = "fallback.properties";

    /**
     * 是否启用
     */
    private boolean enabled = false;

    /**
     * 本地文件地址
     */
    private String fallbackLocation;

    public String getFallbackLocation() {
        return fallbackLocation;
    }

    public void setFallbackLocation(String fallbackLocation) {
        if (!fallbackLocation.contains(".")) {
            // 如果只指定了文件路径，自动拼接文件名
            fallbackLocation = fallbackLocation.endsWith(File.separator) ? fallbackLocation : fallbackLocation + File.separator;
            fallbackLocation += DEFAULT_FILE_NAME;
        }
        this.fallbackLocation = fallbackLocation;
    }

    public boolean isEnabled() {
        return enabled;
    }

    public void setEnabled(boolean enabled) {
        this.enabled = enabled;
    }
}
```

**自动装配实现类**
```java
@Configuration
@EnableConfigurationProperties(ConfigSupportProperties.class)
public class ConfigSupportConfiguration implements ApplicationContextInitializer<ConfigurableApplicationContext>, Ordered {


    private final Logger logger = LoggerFactory.getLogger(getClass());

    /**
     * 重中之重！！！！
     * 一定要注意加载顺序！！！
     *
     * bootstrap.yml 加载类：org.springframework.cloud.bootstrap.config.PropertySourceBootstrapConfiguration 的加载顺序是 
     * HIGHEST_PRECEDENCE+10，
     * 如果当前配置类再其之前加载，无法找到 bootstrap 配置文件中的信息，继而无法加载到本地
     * 所以当前配置类的装配顺序一定要在 PropertySourceBootstrapConfiguration 之后！
     */
    private final Integer orderNumber = Ordered.HIGHEST_PRECEDENCE + 11;

    @Autowired(required = false)
    private List<PropertySourceLocator> propertySourceLocators = Collections.EMPTY_LIST;

    @Autowired
    private ConfigSupportProperties configSupportProperties;

    /**
     * 初始化操作
     */
    @Override
    public void initialize(ConfigurableApplicationContext configurableApplicationContext) {
        // 判断是否开启 config server 管理配置
        if (!isHasCloudConfigLocator(this.propertySourceLocators)) {
            logger.info("Config server 管理配置未启用");
            return;
        }
        logger.info(">>>>>>>>>>>>>>> 检查 config Server 配置资源 <<<<<<<<<<<<<<<");
        ConfigurableEnvironment environment = configurableApplicationContext.getEnvironment();
        MutablePropertySources propertySources = environment.getPropertySources();
        logger.info(">>>>>>>>>>>>> 加载 PropertySources 源：" + propertySources.size() + " 个");

        // 判断配置备份功能是否启用
        if (!configSupportProperties.isEnabled()) {
            logger.info(">>>>>>>>>>>>> 配置备份未启用，使用：{}.enabled 打开 <<<<<<<<<<<<<<", ConfigSupportProperties.CONFIG_PREFIX);
            return;
        }

        if (isCloudConfigLoaded(propertySources)) {
            // 可以从 spring cloud 中获取配置信息
            PropertySource cloudConfigSource = getLoadedCloudPropertySource(propertySources);
            logger.info(">>>>>>>>>>>> 获取 config service 配置资源 <<<<<<<<<<<<<<<");
            Map<String, Object> backupPropertyMap = makeBackupPropertySource(cloudConfigSource);
            doBackup(backupPropertyMap, configSupportProperties.getFallbackLocation());
        } else {
            logger.info(">>>>>>>>>>>>>> 获取 config Server 资源配置失败 <<<<<<<<<<<<<");
            // 不能获取配置信息，从本地读取
            Properties backupProperty = loadBackupProperty(configSupportProperties.getFallbackLocation());
            if (backupProperty != null) {
                Map backupSourceMap = new HashMap<>(backupProperty);
                PropertySource backupSource = new MapPropertySource("backupSource", backupSourceMap);

                propertySources.addFirst(backupSource);
            }
        }

    }


    @Override
    public int getOrder() {
        return orderNumber;
    }


    /**
     * 从本地加载配置
     */
    private Properties loadBackupProperty(String fallbackLocation) {
        logger.info(">>>>>>>>>>>> 正在从本地加载！<<<<<<<<<<<<<<<<<");
        PropertiesFactoryBean propertiesFactoryBean = new PropertiesFactoryBean();
        Properties properties = new Properties();
        try {
            FileSystemResource fileSystemResource = new FileSystemResource(fallbackLocation);
            propertiesFactoryBean.setLocation(fileSystemResource);
            propertiesFactoryBean.afterPropertiesSet();
            properties = propertiesFactoryBean.getObject();
            if (properties != null){
                logger.info(">>>>>>>>>>>>>>> 读取成功！<<<<<<<<<<<<<<<<<<<<<<<<");
            }
        }catch (Exception e){
            e.printStackTrace();
            return null;
        }
        return properties;
    }


    /**
     * 备份配置信息
     */
    private void doBackup(Map<String, Object> backupPropertyMap, String fallbackLocation) {
        FileSystemResource fileSystemResource = new FileSystemResource(fallbackLocation);
        File file = fileSystemResource.getFile();
        try {
            if (!file.exists()){
                file.createNewFile();
            }
            if (!file.canWrite()){
                logger.info(">>>>>>>>>>>> 文件无法写入：{} <<<<<<<<<<<<<<<", fileSystemResource.getPath());
                return;
            }
            Properties properties = new Properties();
            Iterator<String> iterator = backupPropertyMap.keySet().iterator();
            while (iterator.hasNext()) {
                String key = iterator.next();
                properties.setProperty(key, String.valueOf(backupPropertyMap.get(key)));
            }
            FileOutputStream fileOutputStream = new FileOutputStream(fileSystemResource.getFile());
            properties.store(fileOutputStream, "backup cloud config");
        }catch (Exception e){
            logger.info(">>>>>>>>>> 文件操作失败！ <<<<<<<<<<<");
            e.printStackTrace();
        }
    }

    /**
     * 将配置信息转换为 map
     */
    private Map<String, Object> makeBackupPropertySource(PropertySource cloudConfigSource) {
        Map<String, Object> backupSourceMap = new HashMap<>();
        if (cloudConfigSource instanceof CompositePropertySource) {
            CompositePropertySource propertySource = (CompositePropertySource) cloudConfigSource;
            for (PropertySource<?> source : propertySource.getPropertySources()) {
                if (source instanceof MapPropertySource){
                    MapPropertySource mapPropertySource = (MapPropertySource) source;
                    String[] propertyNames = mapPropertySource.getPropertyNames();
                    for (String propertyName : propertyNames) {
                        if (!backupSourceMap.containsKey(propertyName)) {
                            backupSourceMap.put(propertyName, mapPropertySource.getProperty(propertyName));
                        }
                    }
                }
            }
        }

        return backupSourceMap;
    }


    /**
     * config server 管理配置是否开启
     */
    private boolean isHasCloudConfigLocator(List<PropertySourceLocator> propertySourceLocators) {
        for (PropertySourceLocator propertySourceLocator : propertySourceLocators) {
            if (propertySourceLocator instanceof ConfigServicePropertySourceLocator) {
                return true;
            }
        }
        return false;
    }

    /**
     * 获取 config service 配置资源
     */
    private PropertySource getLoadedCloudPropertySource(MutablePropertySources propertySources) {
        if (!propertySources.contains(PropertySourceBootstrapConfiguration.BOOTSTRAP_PROPERTY_SOURCE_NAME)){
            return null;
        }
        PropertySource<?> propertySource = propertySources.get(PropertySourceBootstrapConfiguration.BOOTSTRAP_PROPERTY_SOURCE_NAME);
        if (propertySource instanceof CompositePropertySource) {
            for (PropertySource<?> source : ((CompositePropertySource) propertySource).getPropertySources()) {
                // 如果配置源是 config service，使用此配置源获取配置信息
                // configService 是 bootstrapProperties 加载 spring cloud 的实现：ConfigServiceBootstrapConfiguration
                if ("configService".equals(source.getName())){
                    return source;
                }
            }
        }
        return null;
    }

    /**
     * 判断是否可以从 spring cloud 中获取配置信息
     */
    private boolean isCloudConfigLoaded(MutablePropertySources propertySources) {
        return getLoadedCloudPropertySource(propertySources) != null;
    }
}
```

**META-INF/spring.factories**
```
org.springframework.cloud.bootstrap.BootstrapConfiguration=\
  com.laiyy.gitee.confog.springcloudconfighaclientautoconfig.ConfigSupportConfiguration
```

### 客户端实现

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-client</artifactId>
    </dependency>
    <dependency>
        <groupId>com.laiyy.gitee.confog</groupId>
        <artifactId>spring-cloud-config-ha-client-autoconfig</artifactId>
        <version>0.0.1-SNAPSHOT</version>
    </dependency>
</dependencies>
```

**bootstrap.yml**
```yml
spring:
  cloud:
    config:
      label: master
      uri: http://localhost:9090
      name: config-simple
      profile: dev
      backup:
        enabled: true # 自定义配置 -- 是否启用客户端高可用配置
        fallbackLocation: D:/cloud  # 自动备份的配置文档存放位置
```

**application.yml**
```yml
server:
  port: 9015

spring:
  application:
    name: spring-cloud-config-ha-client-config
```

**启动类**
```java
@SpringBootApplication
@RestController
public class SpringCloudConfigHaClientConfigApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudConfigHaClientConfigApplication.class, args);
    }

    @Value("${com.laiyy.gitee.config}")
    private String config;

    @GetMapping(value = "/config")
    public String getConfig(){
        return config;
    }

}
```

### config server

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-server</artifactId>
    </dependency>
</dependencies>
```

```yml
server:
  port: 9090
spring:
  cloud:
    config:
      server:
        git:
          uri: https://gitee.com/laiyy0728/config-repo.git
          search-paths: config-simple
```

```java
@EnableConfigServer
@SpringBootApplication
public class SpringCloudConfigHaClientConfigServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudConfigHaClientConfigServerApplication.class, args);
    }

}
```

## 验证

先后启动 `config-server`、`config-client`，查看`config-client`控制台输出：
```
Fetching config from server at : http://localhost:9090
Located environment: name=config-simple, profiles=[dev], label=master, version=ee39bf20c492b27c2d1b1d0ff378ad721e79a758, state=null
Located property source: CompositePropertySource {name='configService', propertySources=[MapPropertySource {name='configClient'}, MapPropertySource {name='https://gitee.com/laiyy0728/config-repo.git/config-simple/config-simple-dev.yml'}]}
>>>>>>>>>>>>>>> 检查 config Server 配置资源 <<<<<<<<<<<<<<<
>>>>>>>>>>>>> 加载 PropertySources 源：11 个
>>>>>>>>>>>> 获取 config service 配置资源 <<<<<<<<<<<<<<<
No active profile set, falling back to default profiles: default
```

查看 `d:/cloud`，可见存在 `fallback.properties` 文件，打开文件，可见配置信息如下：
```properties
#backup cloud config
#Wed Apr 10 14:49:36 CST 2019
config.client.version=ee39bf20c492b27c2d1b1d0ff378ad721e79a758
com.laiyy.gitee.config=dev \u73AF\u5883\uFF0Cgit \u7248 spring cloud config-----\!
```

访问 http://localhost:9015/config ，可见打印信息如下：
![config-client-ha-result](/images/spring-cloud/config/client-ha-result.png)

---

停止 `server`、`client`，删除 `d:/cloud/fallback.properties`，将 `ConfigSupportConfiguration` 的 orderNumber 改为 `Ordered.HIGHEST_PRECEDENCE + 9`，再次先后启动 `config-server`、`config-client`，查看控制 `client` 控制台输出如下：
```
>>>>>>>>>>>>>>> 检查 config Server 配置资源 <<<<<<<<<<<<<<<
>>>>>>>>>>>>> 加载 PropertySources 源：10 个
>>>>>>>>>>>>>> 获取 config Server 资源配置失败 <<<<<<<<<<<<<
Fetching config from server at : http://localhost:9090
```

可见，`PropertySources` 源从原来的 11 个，变为 10 个。原因是 `bootstrap.yml` 的加载顺序问题。
在源码：`org.springframework.cloud.bootstrap.config.PropertySourceBootstrapConfiguration` 中，其加载顺序为：`Ordered.HIGHEST_PRECEDENCE + 10`，而 `ConfigSupportConfiguration` 的加载顺序为 `Ordered.HIGHEST_PRECEDENCE + 9`，先于 bootstrap.yml 配置文件加载执行，所以无法获取到远程配置信息，继而无法备份配置信息。

--- 

重新进行第一步验证，然后将 `config-server`、`config-client` 停掉后，只启动 `config-client`，可见其控制台打印信息如下：
```
>>>>>>>>>>>>>>> 检查 config Server 配置资源 <<<<<<<<<<<<<<<
>>>>>>>>>>>>> 加载 PropertySources 源：10 个
>>>>>>>>>>>>>> 获取 config Server 资源配置失败 <<<<<<<<<<<<<
>>>>>>>>>>>> 正在从本地加载！<<<<<<<<<<<<<<<<<
>>>>>>>>>>>>>>> 读取成功！<<<<<<<<<<<<<<<<<<<<<<<<
```

访问 http://localhost:9015/config 正常返回信息。

由此验证`客户端高可用`成功

---

# 服务端高可用

服务端高可用，一般情况下是通过与注册中心结合实现。通过 Ribbon 的负载均衡选择 Config Server 进行连接，来获取配置信息。

***源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-config/spring-cloud-config-ha/spring-cloud-config-ha-server***

eureka 选择使用 `spring-cloud-eureka-server-simple`

## config server
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-server</artifactId>
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
  application:
    name: spring-cloud-config-ha-server-app
server:
  port: 9090
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

```java
@SpringBootApplication
@EnableConfigServer
@EnableDiscoveryClient
public class SpringCloudConfigHaServerConfigApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudConfigHaServerConfigApplication.class, args);
    }

}
```

## config client
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-client</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
</dependencies>
```

**application.yml**
```yml
server:
  port: 9016

spring:
  application:
    name: spring-cloud-config-ha-server-client
```

**bootstrap.yml**
```yml
spring:
  cloud:
    config:
      label: master
      name: config-simple
      profile: dev
      discovery:
        enabled: true # 是否从注册中心获取 config server
        service-id: spring-cloud-config-ha-server-app # 注册中心 config server 的 serviceId
eureka:
  client:
    service-url:
      defauleZone: http://localhost:8761/eureka/
```

```java
@SpringBootApplication
@RestController
public class SpringCloudConfigHaServerClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudConfigHaServerClientApplication.class, args);
    }

    @Value("${com.laiyy.gitee.config}")
    private String config;

    @GetMapping(value = "/config")
    public String getConfig(){
        return config;
    }

}
```

启用验证：访问 http://localhost:9016/config ,返回值如下：
![config-client-ha-result](/images/spring-cloud/config/client-ha-result.png)