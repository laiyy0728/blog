---
title: RocketMQ（1） 环境搭建、基础运行
date: 2019-04-21 16:44:07
updated: 2019-04-21 16:44:07
categories: rocketmq
tags: [RocketMQ, MQ]
---

MQ 全称为 `Message Queue`，是一种应用程序程序对应用程序的通信方式，应用程序通过读写出入队列的消息来通信，而无需专用连接来连接它们。消息传递指的是程序之间通过在消息中发送数据来进行通信，而不是通过直接调用来通信，直接调用通常用于诸如远程过程调用的技术。

<!-- more -->

---

# 主流 MQ 对比

主流 MQ 有 Kafka、RocketMQ、RabbitMQ 等

## Kafka

Kafka 是 Apache 的一个子项目，使用 Scala 实现的一个高性能分布式 publish/subscribe 消息队列系统，主要特点：
- 快速持久化：通过磁盘顺序读写与零拷贝机制，可以在 O(1) 的系统开销下进行消息持久化
- 高吞吐：在一台普通的服务器上可以达到 10W/S 的吞吐量
- 高堆积：支持 Topic 下消费者长时间离线，消息堆积量大
- 完全的分布式系统，Broker、Producer、Consumer 都原生支持分布式，依赖 ZK 实现负载均衡
- 支持 Hadoop 数据并行加载；对于像 Hadoop 一样的日志数据和离线分析系统，但又要求实时处理的限制，是一个可行的解决方案

## RocketMQ
前身是 Metaq，3.0 版本更名为 RocketMQ，alibaba 出品，现交 Apache 孵化。RocketMQ 是一款分布式、队列模型的消息中间件，特点：
- 能够保证严格的消息顺序
- 提供丰富的消息拉取模式
- 高效的订阅者水平扩展能力
- 实时的消息订阅机制
- 支持事务消息
- 亿级消息堆积能力

## RabbitMQ

使用 Erlang 编写的一个开源的消息队列，本身支持：AMQP、XMPP、SMTP、STOMP 等协议，是一个重量级消息队列，更适合企业级开发。同时也实现的 broker 架构，生产者不会讲消息直接发送给队列，消息在发送给客户端时，现在中心队列排队。
对路由、负载均衡、数据持久化有很好的支持。

---

# RocketMQ 单机环境搭建

## 版本

- JDK 版本：1.8+
- RocketMQ：4.4.0
- Maven：3.x
- os：CentOS 6.5 x64

## 单机版环境搭建

1. 解压 RocketMQ 4.4.0 到指定文件夹，并修改解压后的文件夹名称
```
unzip rocketmq-all-4.4.0-bin-release.zip -d /usr/local/include/mq/
cd /usr/local/include/mq/
mv rocketmq-all-4.4.0-bin-release/ rocketmq
```

2. 创建日志、数据文件夹
```
mkdir logs store && cd store
mkdir commitlog consumequeue index
```

3. 修改 `conf/2m-2s-async/broker-a.properties`
```properties
# 所属集群名称
brokerClusterName=rocketmq-cluster
# brijer 名称，不同的配置文件，名称不一样
brokerName=broker-a
# 0 表示 master，大于0表示 salve
brokerId=0
# nameServer 地址，分号分割
namesrvAddr=192.168.52.200:9876
# 在发送消息时，自动创建服务器不存在的 Topic，默认创建的队列数
defaultTopicQueueNums=4
# 是否允许 Broker 自动创建 Topic，生产环境需关闭
autoCreateTopicEnable=true
# 是否允许 Broker 自动创建订阅组，生产环境需关闭
autoCreateSubscriptionGroup=true
# Broker 对外服务的监听端口
listenPort=10911
# 删除文件的时间点，默认是凌晨4点
deleteWhen=04
# 文件保留时间，默认 48 小时
fileReservedTime=48
# commitLog 每个文件的大小，默认 1G
mapedFileSizeCommitLog=1073741824
# consumeQueue 每个文件默认存 30W 条，根据需求调整
mapedFileSizeConsumeQueue=300000
# 检测屋里文件磁盘空间
diskMaxUsedSpaceRatio=88
# 存储路径
storePathRootDir=/usr/local/include/mq/rocketmq/store
# commitLog 存储路径
storePathCommitLog=/usr/local/include/mq/rocketmq/store/commitlog
# 消息队列存储路径
storePathConsumeQueue=/usr/local/include/mq/rocketmq/store/consumequeue
# 消息索引存储路径
storePathIndex=/usr/local/include/mq/rocketmq/store/index
# checkpoint 文件存储路径
storeCheckPoint=/usr/local/include/mq/rocketmq/store/checkpoint
# abort 文件存储路径
abortFile=/usr/local/include/mq/rocketmq/store/abort
# 限制消息大小
maxMessageSize=65535
# broker 角色
# 1、ASYNC_MASTER：异步复制的 Master
# 2、SYNC_MASTER：同步双鞋 Master
# 3、SLAVE：从
brokerRole=ASYNC_MASTER
# 刷盘方式
# 1、ASYNC_FLUSH：异步刷盘
# 2、SYNC_FLUSH：同步刷盘
flushDiskType=ASYNC_FLUSH


#checkTransactionMessageEnable=false

# 发送消息的线程数量
# sendMessageThreadPoolNums=128
# 拉取消息线程池数量
# pullMessageThreadPoolNums=128
```

4. 修改 `conf` 下所有的 xml 文件，将 xml 中的 `${user.home}` 修改为 rocketmq 目录
```
sed -i 's#${user.home}#/usr/local/include/mq/rocketmq#g' *.xml
```

5. 修改 `bin/runbroker.sh`、`bin/runserver.sh` 中的 JVM 参数
![runbroker.sh](/images/rocketmq/runbroker.sh.png)
![runserver.sh](/images/rocketmq/runserver.sh.png)

6. 启动 broker
```
nohup sh ./bin/mqnamesrv &
```

7. 使用 `conf/2m-2s-async/broker-a.properties` 配置文件，启动 broker
```
nohup sh ./bin/mqbroker -c /usr/local/include/mq/rocketmq/conf/2m-2s-async/broker-a.properties > /dev/null 2>&1 &
```

8. 使用 jps 查看启动结果
![jps](/images/rocketmq/jps.png)

## RocketMQ 控制台搭建

下载地址：https://github.com/apache/rocketmq-externals.git master 分支，拉取到本地，使用 IDE 打开，修改：`rocketmq-externals-master\rocketmq-console\src\main\resources\application.properties` 配置文件，指定 RocketMQ nameserver 地址(默认端口为 9876)：
```
rocketmq.config.namesrvAddr=192.168.52.200:9876
```

访问 http://localhost:8080
![rocketmq-console](/images/rocketmq/console.png)

---

# 消息的生产、消费

## 一个简单的消息生产者

使用 SpringBoot 搭建一个简单的消息生产者：
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.apache.rocketmq</groupId>
        <artifactId>rocketmq-client</artifactId>
        <version>${rocketmq.version}</version>
    </dependency>

</dependencies>
```

```java
public class Producer {

    public static void main(String[] args) throws MQClientException, UnsupportedEncodingException, RemotingException, InterruptedException, MQBrokerException {
        // 1、创建 DefaultMQProducer
        DefaultMQProducer producer = new DefaultMQProducer("demo-producer");

        // 2、设置 name server
        producer.setNamesrvAddr("192.168.52.200:9876");

        // 3、开启 producer
        producer.start();

        // 4、创建消息
        Message message = new Message("TOPIC_DEMO", "TAG_A", "KEYS_!", "HELLO！".getBytes(RemotingHelper.DEFAULT_CHARSET));

        // 5、发送消息
        SendResult result = producer.send(message);
        System.out.println(result);
        // 6、关闭 producer
        producer.shutdown();
    }
}
```

运行验证控制台打印信息：
```
SendResult [sendStatus=SEND_OK, msgId=C0A80067617C18B4AAC26A932C790000, offsetMsgId=C0A834C800002A9F0000000000000000, messageQueue=MessageQueue [topic=TOPIC_DEMO, brokerName=broker-a, queueId=0], queueOffset=0]
16:40:30.148 [NettyClientSelector_1] INFO RocketmqRemoting - closeChannel: close the connection to remote address[192.168.52.200:10909] result: true
16:40:30.152 [NettyClientSelector_1] INFO RocketmqRemoting - closeChannel: close the connection to remote address[192.168.52.200:9876] result: true
16:40:30.152 [NettyClientSelector_1] INFO RocketmqRemoting - closeChannel: close the connection to remote address[192.168.52.200:10911] result: true
```

查看 rocketmq-console 中 `消息` 选项卡，并选择主题为 `TOPIC_DEMO`：
![TOPIC_DEMO](/images/rocketmq/message-topic-demo.png)

点击 `MESSAGE DETAIL` 查看消息具体内容：
![message detail](/images/rocketmq/message-detail.png)


## 一个简单的消息消费者

依赖与生产者一致

```java
public class Consumer {

    public static void main(String[] args) throws MQClientException {
        // 1、创建 DefaultMQPushConsumer
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("demo-consumer");

        // 2、设置 name server
        consumer.setNamesrvAddr("192.168.52.200:9876");

        // 设置消息拉取最大数
        consumer.setConsumeMessageBatchMaxSize(2);

        // 3、设置 subscribe
        consumer.subscribe("TOPIC_DEMO", // 要消费的主题
                "*" // 过滤规则
        );

        // 4、创建消息监听
        consumer.registerMessageListener(new MessageListenerConcurrently() {
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> list, ConsumeConcurrentlyContext consumeConcurrentlyContext) {
                // 5、获取消息信息
                for (MessageExt msg : list) {
                    // 获取主题
                    String topic = msg.getTopic();
                    // 获取标签
                    String tags = msg.getTags();
                    // 获取信息
                    try {
                        String result = new String(msg.getBody(), RemotingHelper.DEFAULT_CHARSET);
                        System.out.println("Consumer 消费信息：topic：" + topic+ "，tags：" + tags + "，消息体：" + result);
                    } catch (UnsupportedEncodingException e) {
                        e.printStackTrace();
                        return ConsumeConcurrentlyStatus.RECONSUME_LATER;
                    }
                }
                // 6、返回消息读取状态
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });

        // 7、启动消费者（启动后会阻塞）
        consumer.start();
    }
}
```

运行消费者，查看控制台打印信息：
```
Consumer 消费信息：topic：TOPIC_DEMO，tags：TAG_A，消息体：HELLO！
```