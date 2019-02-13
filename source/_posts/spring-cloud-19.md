---
title: Spring Cloud 微服务（19） --- Actuator(二) <BR> 运行时度量
date: 2019-02-13 13:53:28
updated: 2019-02-13 13:53:28
categories:
    Java
tags:
    - SpringBoot
    - Actuator
---

运行时度量端点，包括： /metrics、/trace、/threaddump、/health 等

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


