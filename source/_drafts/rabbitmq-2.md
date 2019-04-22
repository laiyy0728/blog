---
title: RabbitMQ（2） 基础使用
date: 2019-04-22 22:54:01
updated: 2019-04-22 22:54:01
categories:
tags:
---

RabbitMQ 队列模式分为 6 种模式：`简单队列`、`工作队列`、`发布订阅`、`路由`、`Topic`、`RPC`

<!-- more -->

---

# 简单队列

![simple queue](/images/rabbitmq/simple-queue.png)

其中：`P` 代表 producer，消息生产者。`C` 代表 consumer，消息的消费者。红色的管道代表 `队列`

