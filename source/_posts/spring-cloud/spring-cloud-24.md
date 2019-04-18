---
title: Spring Cloud 微服务（24） --- Zuul(五) <BR> 动态路由、灰度发布
date: 2019-02-20 16:54:01
updated: 2019-02-20 16:54:01
categories:
    spring-cloud
tags: [SpringCloud, Zuul]
---

在了解了动态路由的改造原理、方式后，就可以自实现一个小 demo。可以使用 mysql 作为持久化方式，目的是方面、易于管理。

<!-- more -->

# 动态路由实战

***源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-zuul/spring-cloud-dynamic-route-zuul-server***

## Zuul Server

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
    </dependency>
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
</dependencies>
```

```yml
spring:
  application:
    name: spring-cloud-dynamic-route-zuul-server
  datasource:
    url: jdbc:mysql://localhost:3306/springcloud?useUnicode=true&characterEncoding=utf-8&serverTimezone=Hongkong
    driver-class-name: com.mysql.cj.jdbc.Driver
    username: root
    password: 123456
  jpa:
    hibernate:
      ddl-auto: update
server:
  port: 5555
eureka:
  instance:
    instance-id: ${spring.application.name}:${server.port}
    prefer-ip-address: true
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

```java
// zuul 路由实体
@Entity
@Table(name = "zuul_route")
@Data
public class ZuulRouteEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private int id;

    private String path;

    @Column(name = "service_id")
    private String serviceId;
    private String url;

    @Column(name = "strip_prefix")
    private boolean stripPrefix = true;
    private boolean retryable;
    private boolean enabled;
    private String description;
}


// dao
public interface ZuulPropertiesDao extends JpaRepository<ZuulRouteEntity, Integer> {

    @Query("FROM ZuulRouteEntity WHERE enabled = TRUE")
    List<ZuulRouteEntity> findAllByParams();

}


// 动态路由实现
public class DynamicZuulRouteLocator extends SimpleRouteLocator implements RefreshableRouteLocator {

    @Autowired
    private ZuulProperties zuulProperties;

    @Autowired
    private ZuulPropertiesDao zuulPropertiesDao;

    public DynamicZuulRouteLocator(String servletPath, ZuulProperties properties) {
        super(servletPath, properties);
        this.zuulProperties = properties;
    }

    @Override
    public void refresh() {
        doRefresh();
    }

    @Override
    protected Map<String, ZuulProperties.ZuulRoute> locateRoutes() {
        Map<String, ZuulProperties.ZuulRoute> routeMap = new LinkedHashMap<>();
        routeMap.putAll(super.locateRoutes());
        routeMap.putAll(getProperties());
        Map<String, ZuulProperties.ZuulRoute> values = new LinkedHashMap<>();
        routeMap.forEach((path, zuulRoute) -> {
            path = path.startsWith("/") ? path : "/" + path;
            if (StringUtils.hasText(this.zuulProperties.getPrefix())) {
                path = this.zuulProperties.getPrefix() + path;
                path = path.startsWith("/") ? path : "/" + path;
            }
            values.put(path, zuulRoute);
        });
        return values;
    }

    private Map<String, ZuulProperties.ZuulRoute> getProperties() {
        Map<String, ZuulProperties.ZuulRoute> routeMap = new LinkedHashMap<>();
        List<ZuulRouteEntity> list = zuulPropertiesDao.findAllByParams();
        list.forEach(entity -> {
            if (org.apache.commons.lang.StringUtils.isBlank(entity.getPath())) {
                return;
            }
            ZuulProperties.ZuulRoute route = new ZuulProperties.ZuulRoute();
            BeanUtils.copyProperties(entity, route);
            route.setId(String.valueOf(entity.getId()));
            routeMap.put(route.getPath(), route);
        });
        return routeMap;
    }
}


// 注册到 Spring
@Configuration
public class DynamicZuulConfig {

    private final ZuulProperties zuulProperties;

    private final ServerProperties serverProperties;

    @Autowired
    public DynamicZuulConfig(ZuulProperties zuulProperties, ServerProperties serverProperties) {
        this.zuulProperties = zuulProperties;
        this.serverProperties = serverProperties;
    }

    @Bean
    public DynamicZuulRouteLocator dynamicZuulRouteLocator(){
        return new DynamicZuulRouteLocator(serverProperties.getServlet().getContextPath(), zuulProperties);
    }
}

```

## 验证

在数据库中增加三条数据
```sql
INSERT INTO `springcloud`.`zuul_route` (`id`, `description`, `enabled`, `path`, `retryable`, `service_id`, `strip_prefix`, `url`) VALUES ('1', '重定向到百度', '\1', '/baidu/**', '\0', NULL, '\1', 'http://www.baidu.com');
INSERT INTO `springcloud`.`zuul_route` (`id`, `description`, `enabled`, `path`, `retryable`, `service_id`, `strip_prefix`, `url`) VALUES ('2', 'url', '\1', '/client/**', '\0', NULL, '\1', 'http://localhost:8081');
INSERT INTO `springcloud`.`zuul_route` (`id`, `description`, `enabled`, `path`, `retryable`, `service_id`, `strip_prefix`, `url`) VALUES ('3', 'serviceId', '\1', '/client-1/**', '\0', 'client-a', '\1', NULL);
```

访问 http://localhost:5555/baidu/get-result 、http://localhost:5555/client/get-result 、http://localhost:5555/client-a/get-result
![重定向百度](/images/spring-cloud/zuul/dynamic-route-baidu.png)

```
this is provider service! this port is: 8081 headers: [user-agent]: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/71.0.3578.98 Safari/537.36; [cache-control]: no-cache; [postman-token]: a35929aa-9bcf-f5e4-b79a-0e684b1611ed; [token]: E477CA7B8E7CDCDCE3331742544DE9F1; [content-type]: application/x-www-form-urlencoded;charset=UTF-8; [accept]: */*; [accept-encoding]: gzip, deflate, br; [accept-language]: zh-CN,zh;q=0.9; [x-forwarded-host]: localhost:5555; [x-forwarded-proto]: http; [x-forwarded-prefix]: /client; [x-forwarded-port]: 5555; [x-forwarded-for]: 0:0:0:0:0:0:0:1; [host]: localhost:8081; [connection]: Keep-Alive; 
```

```
{
    "timestamp": "2019-02-21T03:15:18.997+0000",
    "status": 404,
    "error": "Not Found",
    "message": "No message available",
    "path": "/client-a/get-result"
}
```

由此可以证明，动态路由配置成功。


---


# 灰度发布

灰度发布是指在系统迭代新功能时的一种平滑过渡的上线发布方式。灰度发布是在原有的系统基础上，额外增加一个新版本，在这个新版本中，有需要验证的功能修改或添加，使用负载均衡器，引入一小部分流量到新版本应用中，如果这个新版本没有出现差错，再平滑地把线上系统或服务一步步替换成新版本，直至全部替换上线结束。


## 灰度发布实现方式

灰度发布可以使用元数据来实现，元数据有两种

- 标准元数据：标准元数据是服务的各种注册信息，如：ip、端口、健康信息、续约信息等，存储于专门为服务开辟的注册表中，用于其他组件取用以实现整个微服务生态
- 自定义元数据：自定义元数据是使用 `eureka.instance.metadata-map.{key}={value}` 配置，其内部实际上是维护了一个 map 来保存子弹元数据信息，可配置再远端服务，随服务一并注册保存在 Eureka 注册表，对微服务生态没有影响。

## 灰度发布实战

***源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-zuul/spring-cloud-zuul-metadata***

### provider
```yml
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
    instance-id: ${spring.application.name}:${server.port}

spring:
  application:
    name: spring-cloud-metadata-provider-service

---
server:
  port: 7070
spring:
  profiles: node1   # 设定 profile，可以使用 mvn spring-boor:run -Dspring.profiles.active=node1 启动，或者在启动类使用   SpringApplication.run(SpringCloudMetadataProviderServiceApplication.class, "--spring.profiles.active=node1"); 启动
eureka:
  instance:
    metadata-map:
      host-mark: running # 设定当前节点的 metadata，zuul server 使用这个标注来进行路由转发
---
spring:
  profiles: node2
server:
  port: 7071
eureka:
  instance:
    metadata-map:
      host-mark: running
---
spring:
  profiles: node3
server:
  port: 7072
eureka:
  instance:
    metadata-map:
      host-mark: gray  # 当前节点是灰度节点
```

```java
@SpringBootApplication
@EnableDiscoveryClient
@RestController
public class SpringCloudMetadataProviderServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudMetadataProviderServiceApplication.class, args);
        // SpringApplication.run(SpringCloudMetadataProviderServiceApplication.class, "--Dspring.profiles.active=node1"); // 非 maven 启动
    }

    @Value("${server.port}")
    private int port;

    @GetMapping(value = "/get-result")
    public String getResult(){
        return "metadata provider service result, port: " + port;
    }
}
```

### Zuul Server

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
    </dependency>

    <!-- 实现通过 metadata 进行灰度路由 -->
    <dependency>
        <groupId>io.jmnarloch</groupId>
        <artifactId>ribbon-discovery-filter-spring-cloud-starter</artifactId>
        <version>2.1.0</version>
    </dependency>
</dependencies>
```

```yml
spring:
  application:
    name: spring-cloud-metadata-zuul-server
server:
  port: 5555
eureka:
  instance:
    instance-id: ${spring.application.name}:${server.port}
    prefer-ip-address: true
  client:
    service-url:
      defautlZone: http://localhost:8761/eureka/
zuul:
  routes:
    spring-cloud-metadata-provider-service:
      path: /provider/**
      serviceId: spring-cloud-metadata-provider-service
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
// 实现灰度的 Filter
public class GrayFilter extends ZuulFilter {

    @Override
    public String filterType() {
        return FilterConstants.PRE_TYPE;
    }

    @Override
    public int filterOrder() {
        return FilterConstants.PRE_DECORATION_FILTER_ORDER - 1;
    }

    @Override
    public boolean shouldFilter() {
        RequestContext context = RequestContext.getCurrentContext();
        return !context.containsKey(FilterConstants.FORWARD_TO_KEY) && !context.containsKey(FilterConstants.SERVICE_ID_KEY);
    }

    @Override
    public Object run() throws ZuulException {
        HttpServletRequest request = RequestContext.getCurrentContext().getRequest();
        String grayMark = request.getHeader("gray_mark");
        if (StringUtils.isNotBlank(grayMark) && StringUtils.equals("enable", grayMark)) {
            RibbonFilterContextHolder.getCurrentContext().add("host-mark", "gray");
        } else {
            RibbonFilterContextHolder.getCurrentContext().add("host-mark", "running");
        }
        return null;
    }
}


@SpringBootApplication
@EnableDiscoveryClient
@EnableZuulProxy
public class SpringCloudMetadataZuulServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudMetadataZuulServerApplication.class, args);
    }

    @Bean
    public GrayFilter grayFilter(){
        return new GrayFilter();
    }

}
```

### 验证

正常访问： http://localhost:5555/provider/get-result ，查看返回值， port 在 7070 和 7071 之间轮询。
![gray running](/images/spring-cloud/zuul/gray-running.png)

启用 gray_mark header，再次访问，发现 port 始终都是 7072，由此验证灰度成功
![gray](/images/spring-cloud/zuul/gray.png)