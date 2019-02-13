---
title: Spring Cloud 微服务（18） --- Actuator(一) <BR> 常见端点、配置明细端点
date: 2019-02-12 16:53:00
updated: 2019-02-12 16:53:00
categories:
    Java
tags:
    - SpringBoot
    - Actuator
---

在 Hystrix Dashboard 中，使用了 /actuator/hystrix.stream，查看正在运行的项目的运行状态。 其中 `/actuator` 代表 SpringBoot 中的 Actuator 模块。该模版提供了很多生产级别的特性，如：监控、度量应用。Actuator 的特性可以通过众多的 REST 端点，远程 shell、JMX 获得。

<!-- more -->

# 常见 Actuator 端点

![Actuator endpoints 端点](/images/spring-cloud/actuator/endpoints.png)

*路径省略前缀：/actuator*

| 路径 | HTTP 动作 | 描述 |
| :-: | :-: | :-: |
| /conditions | GET | 提供了一份自动配置报告，记录哪些自动配置条件通过了，哪些没通过 |
| /configprops | GET | 描述配置属性（包含默认值）如何注入 Bean |
| /caches | GET | 获取所有的 Cachemanager |
| /caches/{cache} | DELETE | 移除某个 CacheManager |
| /beans | GET | 描述应用程序上下文全部 bean，以及他们的关系 |
| /threaddump | GET | 获取线程活动快照 |
| /env | GET | 获取全部环境属性 |
| /env/{toMatch} | GET | 获取指定名称的特定环境的属性值 |
| /health | GET | 报告应用长须的健康指标，由 HealthIndicator 的实现提供 |
| /httptrace |  GET | 提供基本的 HTTP 请求跟踪信息(时间戳、HTTP 头等)|
| /info | GET | 获取应用程序定制信息，由 Info 开头的属性提供 |
| /loggers | GET | 获取 bean 的日志级别 |
| /loggers/{name} | GET | 获取某个 包、类 路径端点的日志级别 |
| /loggers/{name} | POST | 新增某个 包、类 路径端点的日志级别 |
| /mappings | GET | 描述全部的 URL 路径，以及它们和控制器(包含Actuator端点)的映射关系 |
| /metrics | GET | 报告各种应用程序度量信息，如：内存用量、HTTP 请求计数 |
| /metrics/{name} | GET | 根据名称获取度量信息 |
| /shutdown | POST | 关闭应用，要求 endpoints.shutdown.enabled 为 true |
| /scheduledtasks | GET | 获取所有定时任务信息 |


SpringCloud 中，默认开启了 `/actuator/health`、`/actuator/info` 端点，其他端点都屏蔽掉了。如果需要开启，自定义开启 endpoints 即可
```yml
management:
  endpoints:
    web:
      exposure:
        exclude: ['health','info','beans','env']
```
如果要开启全部端点，设置 `exclude: *` 即可

---

# 配置明细端点

引入pom文件，设置 yml
```xml
 <dependencies>
      <dependency>
          <groupId>org.springframework.boot</groupId>
          <artifactId>spring-boot-starter-actuator</artifactId>
      </dependency>
  </dependencies>
```

```yml
management:
  endpoints:
    web:
      exposure:
        include: '*'
```

## beans 端点

使用 /actuator/beans 端点，获取 JSON 格式的 bean 装配信息。访问： http://localhost:8080/actuator/beans
```json
{
	"contexts": {
		"application": {
			"beans": {
				"webEndpointDiscoverer": {    // bean 名称
					"aliases": [],    // bean 别名
					"scope": "singleton",   // bean 作用域
					"type": "org.springframework.boot.actuate.endpoint.web.annotation.WebEndpointDiscoverer",   // bean 类型
					"resource": "class path resource [org/springframework/boot/actuate/conditionsure/endpoint/web/WebEndpointconditionsuration.class]",   // class 文件物理地址
					"dependencies": [   // 当前 bean 注入的 bean 的列表
						"endpointOperationParameterMapper",
						"endpointMediaTypes"
					]
				},
				"parentId": null
			}
		}
	}
}
```


## conditions 端点

/beans 端点可以看到当前 Spring 上下文有哪些 bean， /conditions 端点可以看到为什么有这个 bean
SpringBoot 自动配置构建与 Spring 的条件化配置之上，提供了众多的 `@Conditional` 注解的配置类，根据 `@Conditional` 条件决定是否自动装配这些 bean。
/conditions 端点提供了一个报告，列出了所有条件，根据条件是否通过进行分组

访问 http://localhost:8080/actuator/conditions

```json
{
	"contexts": {
		"application": {
			"positiveMatches": {    // 成功的自动装配
				"AuditEventsEndpointAutoConfiguration#auditEventsEndpoint": [{
						"condition": "OnBeanCondition",   // 装配条件
						"message": "@ConditionalOnBean (types: org.springframework.boot.actuate.audit.AuditEventRepository; SearchStrategy: all) found bean 'auditEventRepository'; @ConditionalOnMissingBean (types: org.springframework.boot.actuate.audit.AuditEventsEndpoint; SearchStrategy: all) did not find any beans"    // 装配信息
					},
					{
						"condition": "OnEnabledEndpointCondition",
						"message": "@ConditionalOnEnabledEndpoint no property management.endpoint.auditevents.enabled found so using endpoint default"
					}
				],
      },
			"negativeMatches": {      // 失败的自动装配
        "RabbitHealthIndicatorAutoConfiguration": {
          "notMatched": [   // 没有匹配到的条件
            {
              "condition": "OnClassCondition",
              "message": "@ConditionalOnClass did not find required class 'org.springframework.amqp.rabbit.core.RabbitTemplate'"
            }],
            "matched": []   // 匹配到的条件
          },
      },
			"unconditionalClasses": [     // 无条件装配
        "org.springframework.boot.actuate.autoconfigure.management.HeapDumpWebEndpointAutoConfiguration"
      ]
		}
	}
}
```


可以看到，在 `negativeMatches` 的 `RabbitHealthIndicatorAutoConfiguration` 自动状态，没有匹配到 `org.springframework.amqp.rabbit.core.RabbitTemplate` 类，所有没有自动装配。

## env 端点

```json
{
	"activeProfiles": [],   // 启用的 profile 
	"propertySources": [{
			"name": "server.ports",   // 端口设置
			"properties": {
				"local.server.port": {
					"value": 8080   // 本地端口
				}
			}
		},
		{
			"name": "servletContextInitParams",   // servlet 上下文初始化参数信息
			"properties": {}
		},
		{
			"name": "systemProperties",     // 系统配置
			"properties": {
        "java.runtime.name": {    // 配置名称
          "value": "Java(TM) SE Runtime Environment"    // 配置值
        }
      }
		},
		{
			"name": "systemEnvironment",    // 系统环境
			"properties": {
        "USERDOMAIN_ROAMINGPROFILE": {    
          "value": "DESKTOP-QMHTL6V",   // 环境名称
          "origin": "System Environment Property \"USERDOMAIN_ROAMINGPROFILE\""   // 环境值
        }
      }
		},
		{
			"name": "applicationConfig: [classpath:/application.yml]",    // 配置文件信息
			"properties": {
				"management.endpoints.web.exposure.include": {      // 配置文件 key
					"value": "*",       // 配置文件 value
					"origin": "class path resource [application.yml]:5:18"    // 位置
				}
			}
		}
	]
}
```

## mappings 端点

mappings 端点可以生成一份 `控制器到端点` 的映射，即 访问路径与控制器 的映射

```json
{
	"contexts": {
		"application": {
			"mappings": {
				"dispatcherServlets": {   // 所有的 DispatcherServlet
					"dispatcherServlet": [{ // DispatcherServlet
							"handler": "ResourceHttpRequestHandler [class path resource [META-INF/resources/], class path resource [resources/], class path resource [static/], class path resource [public/], ServletContext resource [/], class path resource []]",   // 处理器
							"predicate": "/**/favicon.ico",   // 映射路径
							"details": null
						},
						{
							"handler": "public java.lang.Object org.springframework.boot.actuate.endpoint.web.servlet.AbstractWebMvcEndpointHandlerMapping$OperationHandler.handle(javax.servlet.http.HttpServletRequest,java.util.Map<java.lang.String, java.lang.String>)",
							"predicate": "{[/actuator/auditevents],methods=[GET],produces=[application/vnd.spring-boot.actuator.v2+json || application/json]}",   // 断言（包含端点、请求方式、类型等）
							"details": {    // 详细信息
								"handlerMethod": {  // 处理器
									"className": "org.springframework.boot.actuate.endpoint.web.servlet.AbstractWebMvcEndpointHandlerMapping.OperationHandler", // handle 实现类
									"name": "handle", // 处理器名称
									"descriptor": "(Ljavax/servlet/http/HttpServletRequest;Ljava/util/Map;)Ljava/lang/Object;"    // 描述符
								},
								"requestMappingConditions": {   // 请求映射条件
									"consumes": [],   // 入参类型
									"headers": [],    // 入参头
									"methods": [      // 请求方式
										"GET"
									],
									"params": [],     // 参数
									"patterns": [
										"/actuator/auditevents"   // 请求 URL
									],
									"produces": [{    // 返回值类型
											"mediaType": "application/vnd.spring-boot.actuator.v2+json",
											"negated": false
										},
										{
											"mediaType": "application/json",
											"negated": false
										}
									]
								}
							}
						}
					]
				},
				"servletFilters": [{    // 所有 Filters
						"servletNameMappings": [],    // servlet 名称映射
						"urlPatternMappings": [
							"/*"      // filter 过滤路径
						],
						"name": "webMvcMetricsFilter",    // filter 名称
						"className": "org.springframework.boot.actuate.metrics.web.servlet.WebMvcMetricsFilter"   // Filter 实现类
					}
				],
				"servlets": [{    // 所有 Servlet
						"mappings": [],   // 映射
						"name": "default",    // servlet 名称
						"className": "org.apache.catalina.servlets.DefaultServlet"    // servlet 实现类
					},
					{
						"mappings": [
							"/"
						],
						"name": "dispatcherServlet",
						"className": "org.springframework.web.servlet.DispatcherServlet"
					}
				]
			},
			"parentId": null
		}
	}
}
```
