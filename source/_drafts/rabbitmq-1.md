---
title: RabbitMQ（1） Docker 启动 RabbitMQ
categories: rabbitmq
tags:
  - RabbitMQ
  - MQ
date: 2019-04-22 21:24:52
updated: 2019-04-22 21:24:52
---



`RabbitMQ`是实现了高级消息队列协议（`AMQP`）的开源消息代理软件（亦称面向消息的中间件）。`RabbitMQ`服务器是用Erlang语言编写的，而群集和故障转移是构建在开放电信平台框架上的。所有主要的编程语言均有与代理接口通讯的客户端库。

<!-- more -->

---

# 安装 Docker(CentOS 7)

## 删除旧版本 Docker

```
$ yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-selinux \
                  docker-engine-selinux \
                  docker-engine
```

## 安装 Docker 依赖包

```
$ yum install -y yum-utils \
           device-mapper-persistent-data \
           lvm2
```

## yum 添加 docker 缓存

```
$ yum-config-manager \
    --add-repo \
    https://mirrors.ustc.edu.cn/docker-ce/linux/centos/docker-ce.repo
```

## 安装 docker

```
$ yum makecache fast
$ yum install docker-ce
```

## 设置 docker 开机启动，并启动 docker

```
$ systemctl enable docker
$ systemctl start docker
```

## 测试 docker 是否安装成功 `$ docker run hello-world`

如果出现以下打印信息，则安装成功
```
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
ca4f61b1923c: Pull complete
Digest: sha256:be0cd392e45be79ffeffa6b05338b98ebb16c87b255f48e297ec7f98e123905c
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://cloud.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/engine/userguide/
```

## 设置国内 docker 镜像加速

打开 `/etc/docker/daemon.json` 文件，如果没有则创建该文件。向文件内添加如下信息：
```json
{
  "registry-mirrors": [
    "https://registry.docker-cn.com"
  ]
}
```

## 启用镜像加速，并重启 docker

```
$ systemctl daemon-reload
$ systemctl restart docker
```

---

# docker 安装 rabbitmq

## 拉取 docker 镜像（需要拉取含有管理控制台的镜像）

```
docker pull rabbitmq:management
```
如果出现如下打印，则拉取成功
```
[root@localhost ~]# docker pull rabbitmq:management
management: Pulling from library/rabbitmq
898c46f3b1a1: Pull complete 
63366dfa0a50: Pull complete 
041d4cd74a92: Pull complete 
6e1bee0f8701: Pull complete 
d258c5276992: Pull complete 
3edd5483d555: Pull complete 
32b61a4caa56: Pull complete 
3432dcdcfc27: Pull complete 
4a3b69c89c54: Pull complete 
85fb990e8d2d: Pull complete 
50d6dbbd3ebd: Pull complete 
d85e9dd09c17: Pull complete 
Digest: sha256:10c32cf7028e828da06fe8eb42d1ea9277a56525ae3edd62c48c0035ed64a0cd
Status: Downloaded newer image for rabbitmq:management
```

## 使用默认配置，启动镜像

`docker run --name rabbitmq rabbitmq:management`

参数解释：
> run 代表启动镜像
> --name 给启动的镜像起个名字
> rabbitmq:management 启动哪个镜像

若出现如下打印，则启动成功：
![rabbitmq docker run](/images/rabbitmq/rabbitmq-docker-run.png)
![rabbitmq docker run](/images/rabbitmq/rabbitmq-docker-run-1.png)


## 查看镜像文件 `docker ps -a`

![docker ps -a](/images/rabbitmq/docker-ps-a.png)

此时是访问不了控制台的，原因是 rabbitmq 是在 docker 中启动的，没有映射 docker 到 linux 的端口

## 通过镜像id，停止并删除刚才的镜像

```
docker stop cdd3dadf7e5b
docker rm cdd3dadf7e5b
```

## 映射端口，重新启动 docker

```
docker run -p 5671:5671 -p 5672:5672 -p 4369:4369 -p 25672:25672 -p 15671:15671 -p 15672:15672 --name rabbitmq rabbitmq:management
```

其中： 5671、5672 4369 是 API 端口，15671、15672 是控制台端口。 15672 是浏览器访问控制台的端口。

## 验证安装：

浏览器访问 docker 中的 rabbitmq： http://192.168.52.220:15672 
![rabiitmq dashboard](/images/rabbitmq/rabbitmq-dashboard.png)

默认`用户名、密码`为：`guest/guest`

![rabbitmq-overview](/images/rabbitmq/rabbitmq-overview.png)


---

# rabbitmq 设置

## 进入 `Admin` 选项卡，创建一个新用户

![create-user](/images/rabbitmq/create-user.png)

## 创建一个新的 vhosts

vhosts：virtual hosts 可以理解为“数据库”，以“/” 开头

![create-vhosts](/images/rabbitmq/create-vhosts.png)

## 授权用户访问 vhosts

点击刚创建的 vhosts 名称，进入 vhosts 管理页面，设置可以访问当前 vhosts 的用户
![set user vhosts](/images/rabbitmq/set-user-vhosts.png)

## overview 选项卡

![overview](/images/rabbitmq/overview.png)