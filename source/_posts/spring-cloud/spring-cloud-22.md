---
title: Spring Cloud 微服务（22） --- Zuul(三) <br> Zuul 权限集成
date: 2019-02-18 14:27:07
updated: 2019-02-18 14:27:07
categories:
    spring-cloud
tags:
    - SpringCloud
    - Zuul
---

在原本的单体应用中，通常使用 Apache Shiro、Spring Security 等权限框架，但是在 Spring Cloud 中，面对成千上万的微服务，而且每个服务之间无状态，使用 Shiro、Security 难免力不从心。在解决方案的选择上，传统的单点登录SSO、分布式 session 等，要么致使权限服务器集中化，导致流量臃肿，要么需要实现一套复杂的存储同步机制，都不是最好的解决方案。

<!-- more -->

可以使用 Spring Cloud Zuul 自定义实现权限认证方式

***源码：https://gitee.com/laiyy0728/spring-cloud/tree/master/spring-cloud-zuul/spring-cloud-zuul-security***

---



# 自定义权限认证

## Filter

Zuul 对于请求的转发是通过 Filter 链控制的，可以在 RequestContext 的基础上做任何事。所以只需要在 `spring-cloud-zuul-filter` 的基础上，设置一个执行顺序比较靠前的 Filter，就可以专门用于对请求特定内容做权限认证。

优点：实现灵活度高，可整合已有的权限系统，对原始系统违法化友好
缺点：需要开发一套新的逻辑，维护成本增加，调用链紊乱


## OAuth2.0 + JWT

OAuth2.0 是对于“授权-认证”比较成熟的面向资源的授权协议。整个授权流程中，用户是资源拥有者，服务端需要资源拥有者的授权，这个过程相当于键入密码或者其他第三方登录。触发了这个操作后，客户端就可以向授权服务器申请 Token，拿到后，再携带 Token 到资源所在服务器拉取响应资源。

JWT(JSON Web Token)是一种使用 JSON 格式来规范 Token 或 Session 的协议。由于传统认证方式会生成一个凭证，这个凭证可以是 Token 或 Session，保存于服务端或其他持久化工具中，这样一来，凭证的存取或十分麻烦。JWT 实现了“客户端 Session”。

JWT 的组成部分：
- Header 头部：指定 JWT 使用的签名算法
- Payload 载荷：包含一些自定义与非自定义的认证信息
- Signature：将头部、载荷使用“.”连接后，使用头部的签名算法生成签名信息，并拼装到末尾

OAuth2.0 + JWT 的意义在于，使用 OAuth2.0 协议思想拉取认证生成 TToken，使用 JWT 瞬时保存这个 Token，在客户端与资源端进行对称或非对称加密，是的这个规约具有定时、定量的授权认证功能，从而免去 Token 存储带来的安全或者系统扩展问题。


## 实现

### Zuul Server

```xml
 <dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-oauth2</artifactId>
    </dependency>
</dependencies>
```

```yml
spring:
  application:
    name: spring-cloud-zuul-security-server
server:
  port: 5555
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
    instance-id: ${spring.application.name}:${server.port}
zuul:
  routes:
    spring-cloud-zuul-security-provider-service:
      path: /provider/**
      serviceId: spring-cloud-zuul-security-provider-service
security:
  oauth2:
    client:
      access-token-uri:  http://localhost:7777/uaa/oauth/token # 令牌端点
      user-authorization-uri: http://localhost:7777/uaa/oauth/authorize # 授权端点
      client-id: zuul_server # OAuth2 客户端id
      client-secret: secret # OAuth2 客户端秘钥
    resource:
      jwt:
        key-value: spring-cloud # 使用对称加密，默认算法为 HS256，加密秘钥为 spring-cloud
```

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableZuulProxy
@EnableOAuth2Sso    // 开启 OAuth2.0 sso认证
public class SpringCloudZuulSecurityServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudZuulSecurityServerApplication.class, args);
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                // 声明要鉴权的 urls
                .antMatchers("/login", "/provider/**")
                .permitAll().anyRequest().authenticated().and().csrf().disable();
    }
}
```


### auth server

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-oauth2</artifactId>
    </dependency>
</dependencies>
```

```yml
spring:
  application:
    name: spring-cloud-zuul-security-auth-server
server:
  port: 7777
  servlet:
    context-path: /uaa # web 访问根节点
eureka:
  instance:
    instance-id: ${spring.application.name}:${server.port}
    prefer-ip-address: true
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

```java
// 启动类
@SpringBootApplication
@EnableDiscoveryClient
public class SpringCloudZuulSecurityAuthServerApplication extends WebSecurityConfigurerAdapter {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudZuulSecurityAuthServerApplication.class, args);
    }

    @Bean(name = BeanIds.AUTHENTICATION_MANAGER)  // 设置 Bean 的名称
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }

    @Bean
    public static PasswordEncoder passwordEncoder(){
        // password 编码器
        return NoOpPasswordEncoder.getInstance();
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        // 设置用户和权限
        auth.inMemoryAuthentication().withUser("guest").password("guest").authorities("WRIGHT_READ")
                .and()
                .withUser("admin").password("admin").authorities("WRIGHT_READ", "WRIGHT_WRITE");
    }
}


// OAuth 配置
@Configuration
@EnableAuthorizationServer  // 开启认证服务器
public class OAuthConfiguration extends AuthorizationServerConfigurerAdapter {

    private final AuthenticationManager authenticationManager;

    @Autowired
    public OAuthConfiguration(AuthenticationManager authenticationManager) {
        this.authenticationManager = authenticationManager;
    }

    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        clients.inMemory()
                // 此处的 client 是 zuul server 中 security.oauth2.client.client-id
                .withClient("zuul_server")
                // 此处的 secret 是 zuul server 中 security.oauth2.client.client-secret
                .secret("secret")
                // 作用域
                .scopes("WRIGHT", "READ")
                // 跳过认证确认的过程
                .autoApprove(true)
                // 权限
                .authorities("WRIGHT_READ", "WRIGHT_WRITE")
                // 可以使用的授权类型，默认为空
                // implicit：隐式授权类型
                // refresh_token：刷新令牌获取新的令牌
                // password：资源所有者密码类型
                // authorization_code：授权码类型
                // client_credentials：客户端凭据（客户端ID以及key）类型
                .authorizedGrantTypes("implicit", "refresh_token", "password", "authorization_code");
    }

    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        endpoints.tokenStore(jwtTokenStore())
                .tokenEnhancer(jwtAccessTokenConverter())
                .authenticationManager(authenticationManager);
    }

    @Bean
    public TokenStore jwtTokenStore(){
        return new JwtTokenStore(jwtAccessTokenConverter());
    }

    @Bean
    public JwtAccessTokenConverter jwtAccessTokenConverter(){
        JwtAccessTokenConverter jwtAccessTokenConverter = new JwtAccessTokenConverter();
        // 设置秘钥，需要与 zuul_server 中配置的一样
        jwtAccessTokenConverter.setSigningKey("spring-cloud");
        return jwtAccessTokenConverter;
    }
}
```


### provider

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-oauth2</artifactId>
    </dependency>
</dependencies>
```

```yml
server:
  port: 7070
spring:
  application:
    name: spring-cloud-zuul-security-provider-service
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
    instance-id: ${spring.application.name}:${server.port}
```

```java
@SpringBootApplication
@EnableDiscoveryClient
@RestController
@EnableResourceServer
public class SpringCloudZuulSecurityProviderServiceApplication extends ResourceServerConfigurerAdapter {

    private static final Logger LOGGER = LoggerFactory.getLogger(SpringCloudZuulSecurityProviderServiceApplication.class);

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudZuulSecurityProviderServiceApplication.class, args);
    }

    @GetMapping(value = "/test")
    public String test(HttpServletRequest request) {
        LOGGER.info(">>>>>>>>>>>>>>>>>>>>>>>>> header start! <<<<<<<<<<<<<<<<<<<<<<");
        Enumeration<String> headerNames = request.getHeaderNames();
        while (headerNames.hasMoreElements()){
            String header = headerNames.nextElement();
            String value = request.getHeader(header);
            LOGGER.info(">>>>>>>>>>>>>>>> {} : {} <<<<<<<<<<<<<<<<<<<", header, value);
        }
        LOGGER.info(">>>>>>>>>>>>>>>>>>>>>>>>> header end! <<<<<<<<<<<<<<<<<<<<<<");
        return " test!";
    }

    @Override
    public void configure(HttpSecurity http) throws Exception {
        http.csrf().disable().authorizeRequests().antMatchers("/**").authenticated()
                .antMatchers(HttpMethod.GET, "/test")
                .hasAuthority("WRIGHT_READ");
    }

    @Override
    public void configure(ResourceServerSecurityConfigurer resources) throws Exception {
        resources.resourceId("WRIGHT")
                .tokenStore(tokenStore());
    }

    @Bean
    public TokenStore tokenStore(){
        return new JwtTokenStore(jwtAccessTokenConverter());
    }

    @Bean
    protected JwtAccessTokenConverter jwtAccessTokenConverter(){
        JwtAccessTokenConverter converter = new JwtAccessTokenConverter();
        converter.setSigningKey("spring-cloud");
        return converter;
    }
}
```


### 验证

访问 http://localhost:5555/provider/test 页面返回值如下
![oauth2 权限不足](/images/spring-cloud/zuul/oauth2-error.png)

访问 http://localhost:5555/login 将会自动跳转到 http://localhost:7777/uaa/login 使用 `admin/admin` 登录
![oauth2 登录](/images/spring-cloud/zuul/oauth2-login.png)

访问成功后会返回一个 404 页面，这是因为没有配置成功后跳转页面导致的，暂时不管
![oauth2 登录成功](/images/spring-cloud/zuul/oauth2-logined.png)

再次访问 http://localhost:5555/provider/test 页面返回值如下
![oauth2 访问成功](/images/spring-cloud/zuul/oauth2-succeed.png)

同时查看 provider-service，控制台输出如下
```
>>>>>>>>>>>>>>>>>>>>>>>> header start! <<<<<<<<<<<<<<<<<<<<<<
>>>>>>>>>>>>>>> upgrade-insecure-requests : 1 <<<<<<<<<<<<<<<<<<<
>>>>>>>>>>>>>>> user-agent : Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/71.0.3578.98 Safari/537.36 <<<<<<<<<<<<<<<<<<<
>>>>>>>>>>>>>>> dnt : 1 <<<<<<<<<<<<<<<<<<<
>>>>>>>>>>>>>>> accept : text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8 <<<<<<<<<<<<<<<<<<<
>>>>>>>>>>>>>>> accept-language : zh-CN,zh;q=0.9 <<<<<<<<<<<<<<<<<<<
>>>>>>>>>>>>>>> authorization : bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1NTA2MDcxODQsInVzZXJfbmFtZSI6ImFkbWluIiwiYXV0aG9yaXRpZXMiOlsiV1JJR0hUX1dSSVRFIiwiV1JJR0hUX1JFQUQiXSwianRpIjoiZTJjYmNjNDk
tMzE5ZC00NDdhLTlmMWYtZmY0YzI5ZDFmZWM4IiwiY2xpZW50X2lkIjoic3ByaW5nLWNsb3VkLXp1dWwtc2VjdXJpdHktc2VydmVyIiwic2NvcGUiOlsiV1JJR0hUIiwicmVhZCJdfQ.vOibf3j0seQqsJuH66eLi_zU_P3KeiTn07baUx78T5A <<<<<<<<<<<<<<<<<<<
>>>>>>>>>>>>>>> x-forwarded-host : localhost:5555 <<<<<<<<<<<<<<<<<<<
>>>>>>>>>>>>>>> x-forwarded-proto : http <<<<<<<<<<<<<<<<<<<
>>>>>>>>>>>>>>> x-forwarded-prefix : /provider <<<<<<<<<<<<<<<<<<<
>>>>>>>>>>>>>>> host : localhost:5555 <<<<<<<<<<<<<<<<<<<
>>>>>>>>>>>>>>> x-forwarded-port : 5555 <<<<<<<<<<<<<<<<<<<
>>>>>>>>>>>>>>> x-forwarded-for : 0:0:0:0:0:0:0:1 <<<<<<<<<<<<<<<<<<<
>>>>>>>>>>>>>>> accept-encoding : gzip <<<<<<<<<<<<<<<<<<<
>>>>>>>>>>>>>>> content-length : 0 <<<<<<<<<<<<<<<<<<<
>>>>>>>>>>>>>>> connection : Keep-Alive <<<<<<<<<<<<<<<<<<<
>>>>>>>>>>>>>>>>>>>>>>>> header end! <<<<<<<<<<<<<<<<<<<<<<
```

其中，`authorization` 就是 JWT Token，这个 Token 是使用 base64 加密的，将 `authorization` 去掉 `bearer` 后，其余部分按 “.” 分隔，每个部分分别解密
`eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9` 解码后为：
```json
{
	"alg": "HS256",
	"typ": "JWT"
}
```
`eyJleHAiOjE1NTA2MDcxODQsInVzZXJfbmFtZSI6ImFkbWluIiwiYXV0aG9yaXRpZXMiOlsiV1JJR0hUX1dSSVRFIiwiV1JJR0hUX1JFQUQiXSwianRpIjoiZTJjYmNjNDktMzE5ZC00NDdhLTlmMWYtZmY0YzI5ZDFmZWM4IiwiY2xpZW50X2lkIjoic3ByaW5nLWNsb3VkLXp1dWwtc2VjdXJpdHktc2VydmVyIiwic2NvcGUiOlsiV1JJR0hUIiwicmVhZCJdfQ` 解码后为：
```json
{
	"exp": 1550607184,
	"user_name": "admin",
	"authorities": ["WRIGHT_WRITE", "WRIGHT_READ"],
	"jti": "e2cbcc49-319d-447a-9f1f-ff4c29d1fec8",
	"client_id": "spring-cloud-zuul-security-server",
	"scope": ["WRIGHT", "read"]
}
```
vOibf3j0seQqsJuH66eLi_zU_P3KeiTn07baUx78T5A：这一部分是密文，不能使用 base64 解密