---
title: hadoop（18） YARN <BR /> 资源调度器
date: 2019-12-16 15:33:32
updated: 2019-12-16 15:33:32
categories:
    [hadoop, yarn]
tags:
    [hadoop, yarn]
---


Yarn 是一个资源调度平台，负责为运算程序提供服务器运算资源，相当于一个分布式的 `操作系统`，而 MapReduce 等运算程序相当于 `操作系统上的应用程序`

<!-- more -->

# 基础架构

Yarn 由 `ResourceManager`、`NodeManager`、`ApplicationMaster`、`Container` 等组件构成。

> ResourceManager

1. 处理客户端请求
2. 监控 NodeManager
3. 启动或监控 ApplicationMaster
4. 资源的分配和调度

> NodeManager

1. 管理单个节点上的资源
2. 处理来自 ResourceManager 的命令
3. 处理来自 ApplicationMaster 的命令