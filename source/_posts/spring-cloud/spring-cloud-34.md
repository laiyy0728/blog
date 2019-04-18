---
title: Spring Cloud 微服务（34） --- APM(三) <BR> Pinpoint
date: 2019-04-17 16:38:55
updated: 2019-04-17 16:38:55
categories: spring-cloud
tags: [SpringCloud, APM]
---

`Pinpoint` 是韩国人编写的 APM 系统，是一个分析大规模分布式系统的平台，并提供处理大量跟踪数据的解决方案。


<!-- more -->

# Pinpoint

## 特点

- 分布式事务追踪，跟踪跨分布式应用的消息
- 自动检测应用拓展
- 水平扩展，以便支持大规模服务器集群
- 提供代码级了践行，便于定位失败点和瓶颈
- 提供字节码增强技术，添加新功能无需修改代码

## 优势

- 非侵入式：使用字节码增强技术，添加新功能无需修改代码
- 资源消耗小：对性能影响最小（资源使用量增加约3%）

## 架构模块

- HBase：主要用于存储数据
- Pinpoint Collector：部署在 Web 容器上
- Pinpoint Web：部署在 Web 容器上
- Pinpoint Agent：附加到用于分析的 Java 应用程序

流程：首先通过 agent 收集调用应用的数据，将数据发送到 collector，collector 通过处理和分析数据，最后存储到 HBase 中，可以通过 Pinpoint Web UI 查看已经分析好的调用分析数据

## 数据结构

- Span：RPC 跟踪的基本单位，表示 RPC 到达时处理的工作，包含跟踪数据。Span 将子项标记未 SpanEvent，作为数据结构，每个 Span 包含一个 TraceId
- Trace：一系列跨度，由相关的 RPC(Span) 组成。同一跟踪中的跨距共享相同的 TransactionId。Trace 通过 SpanIds 和 ParentSpanIds 排序为分层树结构
- TraceId：由 TransactionId、SpanId、ParentSpanId 组成的秘钥集合。`TransactionId` 代表消息id，SpanId 和 ParentSpanId 表示 RPC 父子关系
> TransactionId：来自单个事务的分布式系统发送、接收的消息id，必须在整个服务器组是全局唯一的
> SpanId：接收 RPC 消息时处理的作业 ID，是在 RPC 到达节点时生成的
> ParentSpanId：生成 RPC 的父 span 的 spanId，如果节点是事务的起始点，不会有父跨度。

![pinpont 架构图](/images/spring-cloud/apm/pinpoint.png)

## 兼容性

### JDK 兼容性
| Pinpoint 版本 | Agent 需要的 JDK 版本 | Collector 需要的 JDK 版本 | Web 需要的 JDK 版本 |
| :-: | :-: | :-: | :-: |
| 1.0.x | 6-8 | 6+ | 6+ |
| 1.1.x | 6-8 | 7+ | 7+ |
| 1.5.x | 6-8 | 7+ | 7+ |
| 1.6.x | 6-8 | 7+ | 7+ |
| 1.7.x | 6-8 | 8+ | 8+ |
| 1.8.x | 6-8,9+ | 8+ | 8+ |


### Base 兼容性

| Pinpoint 版本 | HBase 0.94.x | HBase 0.98.x | HBase 1.0.x | HBase 1.1.x | HBase 1.2.x |
| :-: | :-: |  :-: |  :-: |  :-: |  :-: | 
| 1.0.x | √ | × | × | × | × |
| 1.1.x | × | not tested | √ | not tested | not tested |
| 1.5.x | × | not tested | √ | not tested | not tested |
| 1.6.x | × | not tested | not tested | not tested | √ |
| 1.7.x | × | not tested | not tested | not tested | √ |
| 1.8.x | × | not tested | not tested | not tested | √ |

### Agent-Collector 兼容性

| Agent 版本 | Collector 1.0.x | Collector 1.1.x | Collector 1.5.x | Collector 1.6.x | Collector 1.7.x | Collector 1.8.x |
| :-: |:-: |:-: |:-: |:-: |:-: |:-: |
| 1.0.x | √ | √ | √ | √ | √ | √ |
| 1.1.x | not tested | √ | √ | √ | √ | √ | 
| 1.5.x | × | × | √ | √ | √ | √ |
| 1.6.x | × | × | not tested | √ | √ | √ |
| 1.7.x | × | × | × | × | √ | √ |
| 1.8.x | × | × | × | × | × | √ |

### Flink 兼容性

| Pinpoint 版本 | flink 1.3.x | flink 1.4.x |
| :-: | :-: | :-: |
| 1.7.x | √ | × |

---

# 实例

HBase 版本为 1.2.11，下载地址：http://mirrors.hust.edu.cn/apache/hbase/hbase-1.2.11/
Pinpoint 版本为 1.7.3，下载地址：https://github.com/naver/pinpoint/releases/tag/1.7.
Tomcat 版本为：8.x

其中，Pinpoint 需要下载 `agent`、`collector`、`web` 三个文件。

## HBase


### 启动 HBase

解压 HBase，修改 `config/hbase-env.sh` 中 JAVA 目录
![HBase Java home](/images/spring-cloud/apm/hbase-havahome.png)

修改后启动 HBase： `./bin/start-hbase.sh`
```
starting master, logging to /opt/hbase/bin/../logs/hbase-root-master-localhost.localdomain.out
Java HotSpot(TM) 64-Bit Server VM warning: ignoring option PermSize=128m; support was removed in 8.0
Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=128m; support was removed in 8.0
```

稍等片刻后，`jps` 查看是否启动完成，若出现 `HMaster`，则启动完成：
![hbase setup](/images/spring-cloud/apm/hbase-setup.png)

### 加载 Pinpoint HBase 脚本

在 https://github.com/naver/pinpoint/tree/master/hbase/scripts 中，获取 `hbase-create.hbase`、`hbase-drop.hbase` 文件，新建目录 `hbase-script`，执行脚本：
```
./bin/hbase shell /root/hbase-script/hbase-create.hbase 
```
![HBase Shell](/images/spring-cloud/apm/hbase-shell.png)


## Pinpoint 

### 启动 Collector、Web

将 tomcat 解压为2个包，分别为 `collector`、`web`，删除 tomcat 目录下 webapps 下除 `ROOT` 的文件夹，并删除 `ROOT` 下所有文件。
将 collector、web 分别解压至对应 tomcat 的 ROOT 目录下，解压命令：
```
jar -xvf pinpoint-collector-1.7.3.war
```
分别修改两个 tomcat 的 `config/server.xml` 文件，修改端口 `8005`、`8080`、`8443`、`8009` 端口，然后分别启动两个 tomcat。启动成功后访问 zipkin：http://192.168.67.136:28080/#/main
![zipkin](/images/spring-cloud/apm/zipkin-dashboard.png)

## 配置 Agent

创建四个文件夹：`eureka`、`provider`、`consumer`、`zuul`，并将四个服务移入对应文件夹，解压 agent.tar.gz，将解压后的文件放入四个文件夹：
![pinpoint agent](/images/spring-cloud/apm/pinpoint-agent.png)

配置 agent 中的 `pinpoint.config` 文件，修改 `profiler.collector.ip` 设置为 `pinpoint-collector` 的地址，如果在同一个服务器上，不用修改。
![pinpoint collector ip](/images/spring-cloud/apm/pinpoint-collectorr-ip.png)

可以看到在 `pinpoint.config` 中监听了 `9994`、`9995`、`9996` 端口，这三个端口在 collector 启动后就开启了，默认即可。如果 collector 需要修改端口，需要修改 `$COLLECTOR_TOMCAT_HOME/webapps/ROOT/WEB-INF/classes/pinpoint-collector.properties` 文件。

## 启动服务

参数解释：
`-Dpinpoint.agentId`：表示 agent 的唯一标识
`-Dpinpoint.applicationName`：表示用用名称

eureka
`java -javaagent:/usr/local/src/pinpoint/soft/eureka/pinpoint-agent-1.7.3/pinpoint-bootstrap-1.7.3.jar -Dpinpoint.agentId=eureka-server -Dpinpoint.applicationName=eureka-server -jar spring-cloud-eureka-server-simple-0.0.1-SNAPSHOT.jar`

provider
`java -javaagent:/usr/local/src/pinpoint/soft/eureka/pinpoint-agent-1.7.3/pinpoint-bootstrap-1.7.3.jar -Dpinpoint.agentId=provider -Dpinpoint.applicationName=provider -jar spring-cloud-apm-skywalking-provider-0.0.1-SNAPSHOT.jar `

consumer
`java -javaagent:/usr/local/src/pinpoint/soft/eureka/pinpoint-agent-1.7.3/pinpoint-bootstrap-1.7.3.jar -Dpinpoint.agentId=consumer -Dpinpoint.applicationName=consumer -jar spring-cloud-apm-skywlaking-consumer-0.0.1-SNAPSHOT.jar `

zuul
`java -javaagent:/usr/local/src/pinpoint/soft/eureka/pinpoint-agent-1.7.3/pinpoint-bootstrap-1.7.3.jar -Dpinpoint.agentId=zuul -Dpinpoint.applicationName=zuul -jar spring-cloud-apm-skywalking-zuul-0.0.1-SNAPSHOT.jar -Xms256m -Xmx256m`

成功启动后，访问 pinpoint：http://192.168.67.136:28080/#/main
![pinpoint dashboard](/images/spring-cloud/apm/pinpoint-dashboard.png)

通过 zuul 获取数据：http://192.168.67.136:9020/client/get-info
![pinpoint zuul](/images/spring-cloud/apm/pinpoint-zuul.png)

再次查看 pinpoint，切换到 zuul 选项卡：
![pinpoint error](/images/spring-cloud/apm/pinpoint-error.png)
![pinpoint error](/images/spring-cloud/apm/pinpoint-error-1.png)
红色代表调用失败（第一次调用时需要从 eureka 获取数据，默认超时一秒）。数字代表调用次数

`Inspector`：检查器，可以查看服务的调用信息。点击查看：
![inspector](/images/spring-cloud/apm/pinpoint-inspector.png)

在 `Inspector` 中，Timeline 选项卡显示请求时间段，`information` 选项卡显示当前节点启动的信息，包括：应用名、`agentId`、启动时间等，`Heap Usage` 显示堆使用情况，`JVM/System Cpu Usage` 显示 CPU 使用情况，`Active Thread` 显示线程使用情况。`Response Time` 显示响应时间，`Data Source` 显示数据库使用情况