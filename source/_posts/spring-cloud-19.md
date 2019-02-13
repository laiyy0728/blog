---
title: Spring Cloud 微服务（19） --- Actuator(二) <BR> 运行时度量
date: 2019-02-13 13:53:28
updated: 2019-02-13 13:53:28
categories:
    Java
tags: 
	[SpringCloud, SpringBoot, Actuator]
---

运行时度量端点，包括： /metrics、/trace、/threaddump、/health 等

<!-- more -->

---

# 运行时度量端点

## metrics 端点

### 访问结果

metrics 端点主要用于在项目运行中，查看计数器、度量器等，如：当前可用内存、空闲内存等

访问：http://localhost:8080/actuator/metrics ，可用查看所有的计数器、度量器名称， 访问 http://localhost:8080/actuator/metrics/{name} 可以查看具体信息

http://localhost:8080/actuator/metrics
```json
{
	"names": [
		"jvm.memory.max",
		"jvm.threads.states",
		"jvm.gc.pause",
		"http.server.requests",
		"jvm.gc.memory.promoted"
	]
}
```

http://localhost:8080/actuator/metrics/jvm.memory.max
```json
{
	"name": "jvm.memory.max",   // 名称
	"description": "The maximum amount of memory in bytes that can be used for memory management",    // 介绍
	"baseUnit": "bytes",    // 单位
	"measurements": [{
		"statistic": "VALUE",
		"value": 5579472895     // 大小，单位 bytes
	}],
	"availableTags": [{
			"tag": "area",
			"values": [
				"heap",
				"nonheap"
			]
		},
		{
			"tag": "id",
			"values": [
				"Compressed Class Space",
				"PS Survivor Space",
				"PS Old Gen",
				"Metaspace",
				"PS Eden Space",
				"Code Cache"
			]
		}
	]
}
```

http://localhost:8080/actuator/metrics/http.server.requests
```json
{
	"name": "http.server.requests",
	"description": null,
	"baseUnit": "seconds",   
	"measurements": [{
			"statistic": "COUNT",
			"value": 28   // 发起的请求总数
		},
		{
			"statistic": "TOTAL_TIME",
			"value": 1.9647074519999999   // 总时长
		},
		{
			"statistic": "MAX",
			"value": 0.002769728
		}
	],
	"availableTags": [{
			"tag": "exception",
			"values": [
				"None"
			]
		},
		{
			"tag": "method",
			"values": [
				"GET"
			]
		},
		{
			"tag": "uri",
			"values": [     // 访问过的 uri
				"/actuator/caches",
				"/**/favicon.ico",
				"/actuator/threaddump",
				"/actuator/env/{toMatch}",
				"/actuator/loggers",
				"/actuator/mappings",
				"/actuator/auditevents",
				"/**",
				"/actuator/env",
				"/actuator/metrics/{requiredMetricName}",
				"/actuator",
				"/actuator/beans",
				"/actuator/httptrace",
				"/actuator/loggers/{name}",
				"/actuator/scheduledtasks",
				"/actuator/conditions",
				"/actuator/heapdump",
				"/actuator/metrics"
			]
		},
		{
			"tag": "outcome",
			"values": [
				"CLIENT_ERROR",
				"SUCCESS"
			]
		},
		{
			"tag": "status",
			"values": [
				"404",
				"200"
			]
		}
	]
}
```

### metrics 端点介绍

| 前缀 | 分类 | 介绍 |
| :-: | :-: | :-: |
| jvm.gc.* | 垃圾收集器 | 已经发生过的垃圾收集次数、消耗时间，使用与标记-请求垃圾收集器和并行垃圾收集器`java.lang.management.GarbageCollectorMXBean` |
| jvm.memory.* | 内存相关 | 分配给应用程序的内存数量和空闲的内容数量等 `java.lang.Runtime` | 
| jvm.classes.* | 类加载器 | JVM 类加载器加载与卸载的类的数量 `java.lang.management.ClassLoadingMXBean` |
| process.*、system.cpu | 系统 | 系统信息，如：处理器数量`java.lang.Runtime`、运行时间`java.lang.management.RuntimeMXBean`、平均负载`java.lang.management.OperatingSystemMXBean` |
| jvm.threads.* | JVM 线程池 | JVM 线程、守护线程数量、峰值等 `java.lang.management.ThreadMXBean` |
| tomcat.* | tomcat | tomcat 相关内容 |
| datasource.* | 数据源 | 数据源链接数据等，仅当 Spring 上下文存在 DataSource 才会有这个信息 |


## httptrace 端点

httptrace 端点主要用于报告所有的 web 请求的详细信息，包括：请求方法、路径、时间戳、头信息等。http://localhost:8080/actuator/httptrace/

```json
{
	"traces": [{
		"timestamp": "2019-02-13T06:29:35.039Z",			// 时间戳
		"principal": null,
		"session": null,	
		"request": {		// 请求
			"method": "GET",		// 请求方式
			"uri": "http://localhost:8080/actuator/httptrace/",		// 请求路径
			"headers": {		// 请求 Headers
				"cookie": [ 	
					"yfx_c_g_u_id_10000001=_ck19021309350014873926766957773; yfx_f_l_v_t_10000001=f_t_1550021700485__r_t_1550021700485__v_t_1550021700485__r_c_0"
				]
			},
			"remoteAddress": null		// 远程地址
		},
		"response": {	// 响应
			"status": 200,		// 响应码
			"headers": {		// 响应 Header
				"Content-Type": [
					"application/vnd.spring-boot.actuator.v2+json;charset=UTF-8"
				]
			}
		},
		"timeTaken": 36		// 用时
	}]
}
```

需要注意，/httptrace 端点只能显示最近的 100 个请求信息，其中也包含对 /httptrace 端点自己的请求。

## threaddump 端点

threaddump 端点可以查看的应用程序的每个线程，其中包含线程的阻塞状态、所状态等，http://localhost:8080/actuator/threaddump/

```json
{
	"threads": [{
			"threadName": "DestroyJavaVM",		// 线程名称
			"threadId": 44,		// 线程 id
			"blockedTime": -1,	
			"blockedCount": 0,
			"waitedTime": -1,
			"waitedCount": 0,
			"lockName": null,
			"lockOwnerId": -1,
			"lockOwnerName": null,
			"inNative": false,
			"suspended": false,
			"threadState": "RUNNABLE",	// 线程状态
			"stackTrace": [],		// 跟踪栈
			"lockedMonitors": [],
			"lockedSynchronizers": [],
			"lockInfo": null
		}
	]
}
```


## health 端点

health 端点：查看当前应用程序的运行状态。 http://localhost:8080/actuator/health/
```json
{
	"status": "UP"
}
```

health 端点在特定情况下，会有额外的信息，如：登录状态、数据库状态等

### Spring Boot 自带的监控指示器

| 键 | 健康指示器 | 说明 |
| :-: | :-: | :-: |
| none | ApplicationHealthIndicator | 永远为 UP |
| db | DataSourceHealthIndicator | 如果数据库能连上，则内容为 UP 和数据库类型；否则为 DOWN |
| diskSpace | DiskSpaceHealthIndicator | 如果可用空间大于阈值，则内容为 UP 和可用磁盘内容；否则为 DOWN |
| jms | JmsHealthIndicator | 如果能连上消息代理，则为 UP 和 JMS 提供方名称；否则为 DOWN |
| mail | MailHealthIndicator | 如果能连上邮件服务器，则内容为 UP 个邮件服务器主机、端口；否则为 DOWN |
| mongo | MongoHealthIndicator | 如果能连上 MongoDb 服务器，则内容为 UP 和 MongoDB 服务器版本；否则为 DOWN |
| rabbit | RabbitHealthIndicator | 如果能连上 Rabbit 服务器，则内容为 UP 和 Rabbit 版本号；否则为 DOWN |
| redis | RedisHealthIndicator | 如果能连上 Redis 服务器，则内容为 UP 和 Redis 服务器版本；否则为 DOWN |
| solr | SolrHealthIndicator | 如果能连上 Solr 服务器，则内容为 UP ；否则为 DOWN |


---

# 关闭应用程序

关闭应用程序可以使用 /actuator/shutdown 端点，使用 POST 请求 http://localhost:8080/actuator/shutdown
```json
{
    "timestamp": "2019-02-13T07:46:54.614+0000",
    "status": 404,
    "error": "Not Found",
    "message": "No message available",
    "path": "/actuator/shutdown"
}
```

这是因为为了保护应用程序，shutdown 端点没有打开的原因，需要打开 shutdown 端点
```yml
management:
  endpoint:
    shutdown:
      enabled: true
```

再次请求：
```json
{
    "message": "Shutting down, bye..."
}
```


---

# 获取应用信息

获取应用信息使用 /actuator/info 端点，默认的响应是：
```json
{}
```

可以通过带 info 前缀的属性，向 info 端点的响应增加内容，如：
```yml
info:
  contect:
    email: laiyy0728@gmail.com
    phone: 18888888888
```

再次请求：
```json
{
    "contect": {
        "email": "laiyy0728@gmail.com",
        "phone": 18888888888
    }
}
```