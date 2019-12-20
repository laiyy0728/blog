---
title: Zookeeper（二） <BR/> 原理、Java API 操作
date: 2019-12-20 10:30:43
updated: 2019-12-20 10:30:43
categories:
    [zookeeper]
tags:
    [zookeeper]
---

&nbsp;

<!-- more -->

# Zookeeper 内部原理

## 选举机制

> 半数机制：集群中半数以上的机器存活，集群可用。所以 Zookeeper 适合安装 ***奇数台服务器***

> Zookeeper 虽然在配置文件中没有指定 Master 和 Slave，但是 Zookeeper 工作室，是有一个节点为 Leader，其他为 Follower；Leader 是通过 `内部选举机制临时产生`

### 选举过程

假设有舞台服务器组成 Zookeeper 集群，它们的 id 为 `1~5`，同时，它们都是最新启动的，没有历史数据，所有配置一样。假设服务器依次启动，其选举过程为：

1. 服务器 A 启动，此时只有一台服务器启动，它发出的报文没有任何响应，选举状态为 `LOOKING`
2. 服务器 B 启动，与 A 通信，互相交换自己的选举结果。由于两者都没有历史数据，所以 id 较大的 B 胜出；但是，由于没有超过半数以上的服务器同意（5台服务器，半数以上是3)，所以 A、B 继续保持 `LOOKING`
3. 服务器 C 启动，与 A、B 通信，根据前面的分析，此时 C 是集群中的 Leader。
4. 服务器 D 启动，与 A、B、C 通信，理论上来说应该是 D 为 Leader，但是 C 已经成为了 Leader， 所以 D 只能成为 Follower
5. 服务器 E 启动，同样与 D 一样，只能当 Follower


## 节点类型

> 持久型

客户端和服务器端断开链接后，创建的节点不删除；
Zookeeper 给该节点名称进行顺序编号

创建 ZNode 时，设置顺序标识，ZNode 名称后会附加一个值，顺序号是一个单调递增的计数器，由父节点维护

***注意***：在分布式系统中，顺序号可以被用于为所有的事件进行全局排序，这样客户端可以通过顺序号推断事件的顺序

> 短暂型
 
客户端与服务器端断开链接后，创建的节点自己删除；
Zookeeper 给该节点名称进行顺序编码

## Stat 结构体

使用 `stat` 命令，查看节点状态，返回值如下

```
[zk: localhost:2181(CONNECTED) 0] stat /laiyy
cZxid = 0x200000002
ctime = Fri Dec 20 10:19:03 CST 2019
mZxid = 0x200000011
mtime = Fri Dec 20 10:36:33 CST 2019
pZxid = 0x200000018
cversion = 9
dataVersion = 1
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 5
numChildren = 7
```

> cZxid

每次修改 Zookeeper 状态，都会受到一个 zxid 形式的时间戳，也就是 Zookeeper 的 `事务 id`；
事务 id 是 Zookeeper 中所有修改的总的次序，每个修改都有一个 zxid，如果 `zxid1 < zxid2`，那么说明 zxid1 在 zxid2 之前发生

> ctime

ZNode 被创建时的毫秒数（1970开始）

> mZxid

ZNode 最后一次修改的事务 id

> mtime

最后一次修改的好描述（1970开始）

> pZxid

ZNode 最后更新的子节点事务 id

> cversion

ZNode 子节点变化数，ZNode 子节点修改次数

> dataVersion

ZNode 数据变化号

> aclVersion

ZNode 访问控制列表的变化号

> ephemeralOwner

如果是临时节点，这个是 ZNode 拥有者的 sessionId；如果不是临时节点，则是 0

> dataLength

ZNode 数据长度

> numChildren

ZNode 子节点数


## 监听器原理

> 首先，需要一个 main 线程

> 在 main 线程中创建 Zookeeper 客户端，此时会创建两个线程，一个负责网络通信，一个负责监听

> 通过 connect 线程，将注册的监听事件发送到 Zookeeper

> 在 Zookeeper 的注册监听器列表中将注册的监听事件添加到列表中

> Zookeeper 坚挺到有数据、路径的变化时，就会将这个消息发送给 监听线程

> 监听线程内部调用 process 方法

> 常见的监听：1、监听节点数据变化(get path [watch])；2、监听子节点增减的变化(ls path [watch])

## 写数据流程

> Client 向 Zookeeper 的 `Server1` 上写数据，发送一个写请求

> 如果 `Server1` 不是 Leader，那么 Server1 会把接收到的请求转发给 Leader。

Zookeeper 的 Server 中有一个是 Leader，Leader 会将写请求广播给各个 Server，各个 Server 写成功后就会通知 Leader

> 当 Leader 收到大多数 Server 数据写成功，就说明数据写成功

如果有三个节点的话，只要有两个节点写成功，就认为数据写成功。写成功后，Leader 会告诉 Server1 数据写成功了

> Server1 通知 Client 数据写成功，流程结束


---


# API 操作

## 创建客户端连接

```java
private static final String ZOOKEEPER_HOST_URL = "hadoop02:2181,hadoop03:2181,hadoop04:2181";
private static final int SESSION_TIMEOUT = 2000;
private ZooKeeper client;

@Before
public void init() throws IOException {
    client = new ZooKeeper(ZOOKEEPER_HOST_URL, SESSION_TIMEOUT, new Watcher() {
        @Override
        public void process(WatchedEvent watchedEvent) {
            System.out.println(watchedEvent);       
        }
    });

}

@After
public void close() throws Exception{
    if (null != client){
        client.close();
    }
}

@Test
public void connect(){
    System.out.println(client);
}
```

> 全日制

```log
2019-12-20 11:37:46,542 INFO [org.apache.zookeeper.ZooKeeper] - Client environment:zookeeper.version=3.4.10-39d3a4f269333c922ed3db283be479f9deacaa0f, built on 03/23/2017 10:13 GMT

State:CONNECTING sessionid:0x0 local:null remoteserver:null lastZxid:0 xid:1 sent:0 recv:0 queuedpkts:0 pendingresp:0 queuedevents:0

....
```

## 创建子节点

```java
@Test
public void createNode() throws Exception{
    // 参数解释
    // 参数 1：子节点路径
    // 参数 2：创建子节点时绑定的数据
    // 参数 3：权限
    //       ANYONE_ID_UNSAFE：任何人都可访问
    //       AUTH_IDS：只有创建者才可访问
    //       OPEN_ACL_UNSAFE：开放的 ACL，不安全（常用）
    //       CREATOR_ALL_ACL：授权用户具备权限
    //       READ_ACL_UNSAFE：制度权限
    // 参数 4：节点类型
    //       PERSISTENT：持久型
    //       PERSISTENT_SEQUENTIAL：持久型带节点序号
    //       EPHEMERAL：临时型
    //       EPHEMERAL_SEQUENTIAL：临时型带序号）
    String path = client.create("/laiyy", "test_api".getBytes(StandardCharsets.UTF_8), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
    System.out.println(path);
}
```

```log
2019-12-20 11:47:27,378 INFO [org.apache.zookeeper.ClientCnxn] - Session establishment complete on server hadoop03/192.168.233.132:2181, sessionid = 0x36f2112be2e0003, negotiated timeout = 4000
/laiyy
2019-12-20 11:47:27,388 INFO [org.apache.zookeeper.ZooKeeper] - Session: 0x36f2112be2e0003 closed
```

> 查看服务器

```
[zk: localhost:2181(CONNECTED) 8] ls /
[laiyy, zookeeper]
```

## 获取子节点

```java
@Test
public void getChildren() throws Exception {
    List<String> children = client.getChildren("/", false);
    System.out.println("子节点包括：" + children);
    // 休眠5秒，防止还没有获取到就已经关闭连接
    TimeUnit.SECONDS.sleep(5);
}
```


## 获取子节点，并监听数据变化

修改创建连接中的 Watcher，启动成功后查看日志

```java
client = new ZooKeeper(ZOOKEEPER_HOST_URL, SESSION_TIMEOUT, new Watcher() {
        @Override
        public void process(WatchedEvent watchedEvent) {
            List<String> children = null;
            try {
                System.out.println("------------------- 监控开始 ---------------------");
                children = client.getChildren("/", true);
                for (String child : children) {
                    System.out.println(child);
                }
                System.out.println("------------------- 监控结束 ----------------------");
            } catch (KeeperException | InterruptedException e) {
                e.printStackTrace();
            }

        }
    });
```

```
------------------- 监控开始 ---------------------
laiyy
zookeeper
------------------- 监控结束 ----------------------
```

在服务器中进行创建、删除节点操作：
```
[zk: localhost:2181(CONNECTED) 18] create /test 'test'
Created /test
[zk: localhost:2181(CONNECTED) 19] delete /tes
```

控制台打印：
```
------------------- 监控开始 ---------------------
laiyy
zookeeper
test
------------------- 监控结束 ----------------------
------------------- 监控开始 ---------------------
laiyy
zookeeper
------------------- 监控结束 ----------------------
```

## 获取节点状态

```java
@Test
public void exist() throws Exception {
    Stat stat = client.exists("/laiyy", false);
    // 如果节点不再存，stat 为 null
    System.out.println(stat);
    // 休眠5秒，防止还没有获取到就已经关闭连接
    TimeUnit.SECONDS.sleep(5);
}
```

---

# 监听服务器节点动态上下线

***需求***：主节点可以有多台，可以动态上下线，任意一台客户端都能实时感知主节点服务器的上下线

## 服务端

```java
public class Server {

    private static final String ZOOKEEPER_HOST_URL = "hadoop02:2181,hadoop03:2181,hadoop04:2181";
    private static final int SESSION_TIMEOUT = 2000;
    private ZooKeeper zooKeeper;

    public static void main(String[] args) throws IOException, KeeperException, InterruptedException {
        args = new String[]{"localhost:001"};
        Server server = new Server();
        // 连接集群
        server.connect();
        // 注册节点
        server.register(args[0]);

        TimeUnit.MINUTES.sleep(10);
    }

    private void register(String hostname) throws KeeperException, InterruptedException {
        zooKeeper.create("/servers/server", hostname.getBytes(StandardCharsets.UTF_8), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);
        System.out.println(hostname + " 上线了！");
    }


    private void connect() throws IOException {
        zooKeeper = new ZooKeeper(ZOOKEEPER_HOST_URL, SESSION_TIMEOUT, new Watcher() {
            @Override
            public void process(WatchedEvent watchedEvent) {
                System.out.println(watchedEvent);
            }
        });

    }

}
```

## 客户端监听

```java
public class Client {

    private static final String ZOOKEEPER_HOST_URL = "hadoop02:2181,hadoop03:2181,hadoop04:2181";
    private static final int SESSION_TIMEOUT = 2000;
    private ZooKeeper zooKeeper;

    public static void main(String[] args) throws Exception {
        Client client = new Client();

        client.connect();

        client.getChildren();

        TimeUnit.MINUTES.sleep(5);
    }

    private void getChildren() throws Exception {
        List<String> children =
                zooKeeper.getChildren("/servers", true);

        // 存储服务器节点集合名称
        List<String> hosts = new ArrayList<>();
        for (String child : children) {
            byte[] data = zooKeeper.getData("/servers/" + child, false, null);
            hosts.add(new String(data));
        }

        // 打印主机名称
        for (String host : hosts) {
            System.out.println(" 监听到 " + host);
        }
    }

    private void connect() throws Exception {
        zooKeeper = new ZooKeeper(ZOOKEEPER_HOST_URL, SESSION_TIMEOUT, new Watcher() {
            @Override
            public void process(WatchedEvent watchedEvent) {
                System.out.println(watchedEvent);
                try {
                    getChildren();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        });
    }

}
```


## 测试

> 先在集群上创建一个 `/servers` 节点，然后启动 Client
```log
[zk: localhost:2181(CONNECTED) 20] create /servers 'servers'
Created /servers


# java 控制打印
WatchedEvent state:SyncConnected type:None path:null
```

> 在服务器上创建 `/servers/server` 节点，查看 client 控制台

```log
[zk: localhost:2181(CONNECTED) 21] create -e -s /servers/server 'server'
Created /servers/server0000000000


# java 控制台打印
WatchedEvent state:SyncConnected type:NodeChildrenChanged path:/servers
 监听到 server
```

> 运行 java 的 server，查看 server 和 client 控制台


```log
# server

WatchedEvent state:SyncConnected type:None path:null
localhost:001 上线了！


# client 
WatchedEvent state:SyncConnected type:NodeChildrenChanged path:/servers
 监听到 localhost:001
 监听到 server2
 监听到 server
```

> 关掉 java 的 server，查看 client 输出

```
WatchedEvent state:SyncConnected type:NodeChildrenChanged path:/servers
 监听到 server2
 监听到 server
```


由此可以验证，Client 实现了 server 动态上下线监听