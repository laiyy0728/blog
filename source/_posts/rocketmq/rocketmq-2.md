---
title: RocketMQ（2）  顺序消息、事务消息
date: 2019-04-21 17:29:47
updated: 2019-04-21 17:29:47
categories: rocketmq
tags: [RocketMQ, MQ]
---

RocketMQ 顺序消息：消息有序是指可以按照消息发送顺序来消费。RocketMQ 可以严格的保证消息有序，但是这个顺序逼格不是全局顺序，只是分区(queue)顺序。要保证群居顺序，只能有一个分区。

<!-- more -->

# 顺序消息

在 MQ 模型中，顺序要由三个阶段保证：
- 消息被发送时，保持顺序
- 消息被存储时的顺序和发送的顺序一致
- 消息被消费时的顺序和存储的顺序一致

发送时保持顺序，意味着对于有顺序要求的消息，用户应该在同一个线程中采用同步的方式发送。存储保持和发送的顺序一致，则要求在同一线程中被发送出来的消息 A/B，存储时 A 要在 B 之前。而消费保持和存储一致，则要求消息 A/B 到达 Consumer 之后必须按照先后顺序被处理。

![order](/images/rocketmq/order.png)

## 生产者

```java
package com.laiyy.study.rocketmqprovider.order;

import org.apache.rocketmq.client.exception.MQBrokerException;
import org.apache.rocketmq.client.exception.MQClientException;
import org.apache.rocketmq.client.producer.DefaultMQProducer;
import org.apache.rocketmq.client.producer.MessageQueueSelector;
import org.apache.rocketmq.client.producer.SendResult;
import org.apache.rocketmq.common.message.Message;
import org.apache.rocketmq.common.message.MessageQueue;
import org.apache.rocketmq.remoting.common.RemotingHelper;
import org.apache.rocketmq.remoting.exception.RemotingException;

import java.io.UnsupportedEncodingException;
import java.util.List;

/**
 * @author laiyy
 * @date 2019/4/21 16:18
 * @description
 */
public class OrderProducer {

    public static void main(String[] args) throws MQClientException, UnsupportedEncodingException, RemotingException, InterruptedException, MQBrokerException {
        // 1、创建 DefaultMQProducer
        DefaultMQProducer producer = new DefaultMQProducer("demo-producer");

        // 2、设置 name server
        producer.setNamesrvAddr("192.168.52.200:9876");

        // 3、开启 producer
        producer.start();

        // 连续发送 5 条信息
        for (int index = 1; index <= 5; index++) {
            // 创建消息
            Message message = new Message("TOPIC_DEMO", "TAG_A", "KEYS_!", ("HELLO！" + index).getBytes(RemotingHelper.DEFAULT_CHARSET));

            // 指定 MessageQueue，顺序发送消息
            // 第一个参数：消息体
            // 第二个参数：选中指定的消息队列对象（会将所有的消息队列传进来，需要自己选择）
            // 第三个参数：选择对应的队列下标
            SendResult result = producer.send(message, new MessageQueueSelector() {
                // 第一个参数：所有的消息队列对象
                // 第二个参数：消息体
                // 第三个参数：传入的消息队列下标
                @Override
                public MessageQueue select(List<MessageQueue> list, Message message, Object o) {
                    // 获取队列下标
                    int index = (int) o;
                    return list.get(index);
                }
            }, 0);
            System.out.println("发送第：" + index + " 条信息成功：" + result);
        }
        // 关闭 producer
        producer.shutdown();
    }
}
```

控制台输出结果：
```
发送第：1 条信息成功：SendResult [sendStatus=SEND_OK, msgId=C0A800677E4C18B4AAC26ACE66560000, offsetMsgId=C0A834C800002A9F00000000000000B8, messageQueue=MessageQueue [topic=TOPIC_DEMO, brokerName=broker-a, queueId=0], queueOffset=1]
发送第：2 条信息成功：SendResult [sendStatus=SEND_OK, msgId=C0A800677E4C18B4AAC26ACE66630001, offsetMsgId=C0A834C800002A9F0000000000000171, messageQueue=MessageQueue [topic=TOPIC_DEMO, brokerName=broker-a, queueId=0], queueOffset=2]
发送第：3 条信息成功：SendResult [sendStatus=SEND_OK, msgId=C0A800677E4C18B4AAC26ACE66660002, offsetMsgId=C0A834C800002A9F000000000000022A, messageQueue=MessageQueue [topic=TOPIC_DEMO, brokerName=broker-a, queueId=0], queueOffset=3]
发送第：4 条信息成功：SendResult [sendStatus=SEND_OK, msgId=C0A800677E4C18B4AAC26ACE66690003, offsetMsgId=C0A834C800002A9F00000000000002E3, messageQueue=MessageQueue [topic=TOPIC_DEMO, brokerName=broker-a, queueId=0], queueOffset=4]
发送第：5 条信息成功：SendResult [sendStatus=SEND_OK, msgId=C0A800677E4C18B4AAC26ACE666C0004, offsetMsgId=C0A834C800002A9F000000000000039C, messageQueue=MessageQueue [topic=TOPIC_DEMO, brokerName=broker-a, queueId=0], queueOffset=5]
17:45:11.545 [NettyClientSelector_1] INFO RocketmqRemoting - closeChannel: close the connection to remote address[192.168.52.200:10909] result: true
17:45:11.548 [NettyClientSelector_1] INFO RocketmqRemoting - closeChannel: close the connection to remote address[192.168.52.200:9876] result: true
17:45:11.549 [NettyClientSelector_1] INFO RocketmqRemoting - closeChannel: close the connection to remote address[192.168.52.200:10911] result: true

Process finished with exit code 0
```

可以看到，所有消息的  `queueId` 都为 0，顺序消息生产成功。


## 消费者

```java
public class OrderConsumer {

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
        consumer.registerMessageListener(new MessageListenerOrderly() {
            @Override
            public ConsumeOrderlyStatus consumeMessage(List<MessageExt> list, ConsumeOrderlyContext consumeOrderlyContext) {
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
                        return ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT;
                    }
                }
                // 6、返回消息读取状态
                return ConsumeOrderlyStatus.SUCCESS;
            }
        });
        // 启动消费者
        consumer.start();
    }
}
```

顺序消费者与之前的 demo 最大的不同，在于 `message listener` 从 `MessageListenerConcurrently` 变为 `MessageListenerOrderly`，消费标识从 `ConsumeConcurrentlyStatus` 变为 `ConsumeOrderlyStatus`。

查看控制台输出：
```
Consumer 消费信息：topic：TOPIC_DEMO，tags：TAG_A，消息体：HELLO！1
Consumer 消费信息：topic：TOPIC_DEMO，tags：TAG_A，消息体：HELLO！2
Consumer 消费信息：topic：TOPIC_DEMO，tags：TAG_A，消息体：HELLO！3
Consumer 消费信息：topic：TOPIC_DEMO，tags：TAG_A，消息体：HELLO！4
Consumer 消费信息：topic：TOPIC_DEMO，tags：TAG_A，消息体：HELLO！5
```


---

# 事务消息

在 RocketMQ 4.3 版本后，开放了事务消息。

## RocketMQ 事务消息流程

RocketMQ 的事务消息，只要是通过消息的异步处理，可以保证本地事务和消息发送同事成功执行或失败，从而保证数据的最终一致性。

![Transaction message](/images/rocketmq/transaction-message.png)

MQ 事务消息解决分布式事务问题，但是第三方 MQ 支持事务消息的中间件不多，如 RockctMQ，它们支持事务的方式也是类似于采用二阶段提交，但是市面上一些主流的 MQ 都是不支持事务消息的，如：Kafka、RabbitMQ

以 RocketMQ 为例，事务消息实现思路大致为：
- 第一阶段的 Prepared 消息，会拿到消息的地址
- 第二阶段执行本地事务
- 第三阶段通过第一阶段拿到的地址去访问消息，并修改状态

也就是说，在业务方法内想要消息队列提交两次消息，一次发送消息和一次确认消息。如果确认消息发送失败，RocketMQ 会定期扫描消息集群中的事务消息。这时候发现了 prepared 消息，它会向消息发送者确认，所以生产方需要实现一个 check 接口。RocketMQ 会根据发送端设置的策略来决定是回滚还是继续发送确认消息。这样就保证了消息发送与本地事务同时成功或同时失败。
![Transaction message](/images/rocketmq/transaction-message-1.png)



事务消息的成功投递需要三个 Topic，分别是
- Half Topic：用于记录所有的 prepare 消息
- Op Half Topic：记录以及提交了状态的 prepare 消息
- Real Topic：事务消息真正的 topic，在 commit 后才会将消息写入该 topic，从而进行消息投递。


## 事务消息实现

```java
public class TransactionProducer {

    public static void main(String[] args) throws MQClientException, UnsupportedEncodingException, RemotingException, InterruptedException, MQBrokerException {
        // 1、创建 TransactionMQProducer
        TransactionMQProducer producer = new TransactionMQProducer("transaction-producer");

        // 2、设置 name server
        producer.setNamesrvAddr("192.168.52.200:9876");

        // 3、指定消息监听对象，用于执行本地事务和消息回查
        TransactionListenerImpl transactionListener = new TransactionListenerImpl();
        producer.setTransactionListener(transactionListener);

        // 4、线程池
        ThreadPoolExecutor executor = new ThreadPoolExecutor(2, 5, 100, TimeUnit.SECONDS, new ArrayBlockingQueue<Runnable>(2000), new ThreadFactory() {
            @Override
            public Thread newThread(Runnable r) {
                Thread thread = new Thread(r);
                thread.setName("client-transaction-msg-thread");
                return thread;
            }
        });

        producer.setExecutorService(executor);

        // 5、开启 producer
        producer.start();

        // 6、创建消息
        Message message = new Message("TRANSACTION_TOPIC", "TAG_A", "KEYS_!", "HELLO！TRANSACTION!".getBytes(RemotingHelper.DEFAULT_CHARSET));

        // 7、发送消息
        TransactionSendResult result = producer.sendMessageInTransaction(message, "hello-transaction");

        System.out.println(result);

        // 关闭 producer
        producer.shutdown();
    }

}
```


事务消息监听器：
```java
public class TransactionListenerImpl implements TransactionListener {

    /**
     * 存储对应书屋的状态信息， key：事务id，value：事务执行的状态
     */
    private ConcurrentMap<String, Integer> maps = new ConcurrentHashMap<>();

    /**
     * 执行本地事务
     *
     * @param message
     * @param o
     * @return
     */
    @Override
    public LocalTransactionState executeLocalTransaction(Message message, Object o) {
        // 事务id
        String transactionId = message.getTransactionId();

        // 0：执行中，状态未知
        // 1：本地事务执行成功
        // 2：本地事务执行失败

        maps.put(transactionId, 0);

        try {
            System.out.println("正在执行本地事务。。。。");
            // 模拟本地事务
            TimeUnit.SECONDS.sleep(65);
            System.out.println("本地事务执行成功。。。。");
            maps.put(transactionId, 1);
        } catch (InterruptedException e) {
            e.printStackTrace();
            maps.put(transactionId, 2);
            return LocalTransactionState.ROLLBACK_MESSAGE;
        }

        return LocalTransactionState.COMMIT_MESSAGE;
    }

    /**
     * 消息回查
     *
     * @param messageExt
     * @return
     */
    @Override
    public LocalTransactionState checkLocalTransaction(MessageExt messageExt) {
        String transactionId = messageExt.getTransactionId();

        System.out.println("正在执行消息回查，事务id：" + transactionId);

        // 获取事务id的执行状态
        if (maps.containsKey(transactionId)) {
            int status = maps.get(transactionId);
            System.out.println("消息回查状态：" + status);
            switch (status) {
                case 0:
                    return LocalTransactionState.UNKNOW;
                case 1:
                    return LocalTransactionState.COMMIT_MESSAGE;
                default:
                    return LocalTransactionState.ROLLBACK_MESSAGE;
            }
        }
        return LocalTransactionState.UNKNOW;
    }
}
```

运行生产者，查看控制台输出：
```
正在执行本地事务。。。。
正在执行消息回查，事务id：C0A800678F0818B4AAC26AEDDEB10000
消息回查状态：0
本地事务执行成功。。。。
```

需要注意：消息回查会隔一段时间执行一次，如果执行本地事务的时间太短，则控制台不会输出事务回查日志。


--- 

# 广播消息

## 生产者

```java
public class Producer {

    public static void main(String[] args) throws MQClientException, UnsupportedEncodingException, RemotingException, InterruptedException, MQBrokerException {
        // 1、创建 DefaultMQProducer
        DefaultMQProducer producer = new DefaultMQProducer("boardcast-producer");

        // 2、设置 name server
        producer.setNamesrvAddr("192.168.52.200:9876");

        // 3、开启 producer
        producer.start();

        for (int index = 1; index <= 10; index++) {
            Message message = new Message("BOARD_CAST_TOPIC", "TAG_A", "KEYS_" + index, ("HELLO！" + index).getBytes(RemotingHelper.DEFAULT_CHARSET));
            SendResult result = producer.send(message);
            System.out.println(result);
        }

        // 关闭 producer
        producer.shutdown();
    }

}
```


## 消费者

消费者需要将消费模式修改为 广播消费：  consumer.setMessageModel(MessageModel.BROADCASTING);

```java
public class Consumer {

    public static void main(String[] args) throws MQClientException {
        // 1、创建 DefaultMQPushConsumer
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("boardcast-consumer");

        // 2、设置 name server
        consumer.setNamesrvAddr("192.168.52.200:9876");

        // 设置消息拉取最大数
        consumer.setConsumeMessageBatchMaxSize(2);


        // 修改消费模式，默认是集群消费模式，修改为广播消费模式
        consumer.setMessageModel(MessageModel.BROADCASTING);

        // 3、设置 subscribe
        consumer.subscribe("BOARD_CAST_TOPIC", // 要消费的主题
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
                        System.out.println("A  Consumer 消费信息：topic：" + topic+ "，tags：" + tags + "，消息体：" + result);
                    } catch (UnsupportedEncodingException e) {
                        e.printStackTrace();
                        return ConsumeConcurrentlyStatus.RECONSUME_LATER;
                    }
                }
                // 6、返回消息读取状态
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
        consumer.start();
    }
}
```

## 验证

### 生产者控制台输出
```
SendResult [sendStatus=SEND_OK, msgId=C0A80067971418B4AAC26B2965570000, offsetMsgId=C0A834C800002A9F00000000000026D0, messageQueue=MessageQueue [topic=BOARD_CAST_TOPIC, brokerName=broker-a, queueId=1], queueOffset=0]
SendResult [sendStatus=SEND_OK, msgId=C0A80067971418B4AAC26B2965660001, offsetMsgId=C0A834C800002A9F000000000000278F, messageQueue=MessageQueue [topic=BOARD_CAST_TOPIC, brokerName=broker-a, queueId=2], queueOffset=10]
SendResult [sendStatus=SEND_OK, msgId=C0A80067971418B4AAC26B29656C0002, offsetMsgId=C0A834C800002A9F000000000000284E, messageQueue=MessageQueue [topic=BOARD_CAST_TOPIC, brokerName=broker-a, queueId=3], queueOffset=0]
SendResult [sendStatus=SEND_OK, msgId=C0A80067971418B4AAC26B2965700003, offsetMsgId=C0A834C800002A9F000000000000290D, messageQueue=MessageQueue [topic=BOARD_CAST_TOPIC, brokerName=broker-a, queueId=0], queueOffset=0]
SendResult [sendStatus=SEND_OK, msgId=C0A80067971418B4AAC26B29657B0004, offsetMsgId=C0A834C800002A9F00000000000029CC, messageQueue=MessageQueue [topic=BOARD_CAST_TOPIC, brokerName=broker-a, queueId=1], queueOffset=1]
SendResult [sendStatus=SEND_OK, msgId=C0A80067971418B4AAC26B2965880005, offsetMsgId=C0A834C800002A9F0000000000002A8B, messageQueue=MessageQueue [topic=BOARD_CAST_TOPIC, brokerName=broker-a, queueId=2], queueOffset=11]
SendResult [sendStatus=SEND_OK, msgId=C0A80067971418B4AAC26B29658E0006, offsetMsgId=C0A834C800002A9F0000000000002B4A, messageQueue=MessageQueue [topic=BOARD_CAST_TOPIC, brokerName=broker-a, queueId=3], queueOffset=1]
SendResult [sendStatus=SEND_OK, msgId=C0A80067971418B4AAC26B2965960007, offsetMsgId=C0A834C800002A9F0000000000002C09, messageQueue=MessageQueue [topic=BOARD_CAST_TOPIC, brokerName=broker-a, queueId=0], queueOffset=1]
SendResult [sendStatus=SEND_OK, msgId=C0A80067971418B4AAC26B29659D0008, offsetMsgId=C0A834C800002A9F0000000000002CC8, messageQueue=MessageQueue [topic=BOARD_CAST_TOPIC, brokerName=broker-a, queueId=1], queueOffset=2]
SendResult [sendStatus=SEND_OK, msgId=C0A80067971418B4AAC26B2965AB0009, offsetMsgId=C0A834C800002A9F0000000000002D87, messageQueue=MessageQueue [topic=BOARD_CAST_TOPIC, brokerName=broker-a, queueId=2], queueOffset=12]
19:24:35.135 [NettyClientSelector_1] INFO RocketmqRemoting - closeChannel: close the connection to remote address[192.168.52.200:10911] result: true
19:24:35.140 [NettyClientSelector_1] INFO RocketmqRemoting - closeChannel: close the connection to remote address[192.168.52.200:9876] result: true
19:24:35.140 [NettyClientSelector_1] INFO RocketmqRemoting - closeChannel: close the connection to remote address[192.168.52.200:10909] result: true
```

### 消费者控制台输出

```
A  Consumer 消费信息：topic：BOARD_CAST_TOPIC，tags：TAG_A，消息体：HELLO！1
A  Consumer 消费信息：topic：BOARD_CAST_TOPIC，tags：TAG_A，消息体：HELLO！2
A  Consumer 消费信息：topic：BOARD_CAST_TOPIC，tags：TAG_A，消息体：HELLO！5
A  Consumer 消费信息：topic：BOARD_CAST_TOPIC，tags：TAG_A，消息体：HELLO！4
A  Consumer 消费信息：topic：BOARD_CAST_TOPIC，tags：TAG_A，消息体：HELLO！3
A  Consumer 消费信息：topic：BOARD_CAST_TOPIC，tags：TAG_A，消息体：HELLO！7
A  Consumer 消费信息：topic：BOARD_CAST_TOPIC，tags：TAG_A，消息体：HELLO！6
A  Consumer 消费信息：topic：BOARD_CAST_TOPIC，tags：TAG_A，消息体：HELLO！8
A  Consumer 消费信息：topic：BOARD_CAST_TOPIC，tags：TAG_A，消息体：HELLO！9
A  Consumer 消费信息：topic：BOARD_CAST_TOPIC，tags：TAG_A，消息体：HELLO！10
```

```
B  Consumer 消费信息：topic：BOARD_CAST_TOPIC，tags：TAG_A，消息体：HELLO！1
B  Consumer 消费信息：topic：BOARD_CAST_TOPIC，tags：TAG_A，消息体：HELLO！2
B  Consumer 消费信息：topic：BOARD_CAST_TOPIC，tags：TAG_A，消息体：HELLO！3
B  Consumer 消费信息：topic：BOARD_CAST_TOPIC，tags：TAG_A，消息体：HELLO！5
B  Consumer 消费信息：topic：BOARD_CAST_TOPIC，tags：TAG_A，消息体：HELLO！4
B  Consumer 消费信息：topic：BOARD_CAST_TOPIC，tags：TAG_A，消息体：HELLO！6
B  Consumer 消费信息：topic：BOARD_CAST_TOPIC，tags：TAG_A，消息体：HELLO！7
B  Consumer 消费信息：topic：BOARD_CAST_TOPIC，tags：TAG_A，消息体：HELLO！8
B  Consumer 消费信息：topic：BOARD_CAST_TOPIC，tags：TAG_A，消息体：HELLO！9
B  Consumer 消费信息：topic：BOARD_CAST_TOPIC，tags：TAG_A，消息体：HELLO！10
```