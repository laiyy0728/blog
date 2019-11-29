---
title: Hadoop（6） <br/> HDFS API 操作
date: 2019-11-27 09:24:43
updated: 2019-11-27 09:24:48
categories:
    [hadoop]
tags: [hadoop]
---


此前，完成了一个基础的完全分布式集群，并且使用 Java 程序代码实现测试连通了 Hadoop 集群，且在 HDFS 中创建了一个文件夹。由此开始学习 Hadoop 的一些 Java API 操作。

<!-- more -->

API 操作通用方法

```java
@Before
public void initFileSystem() throws URISyntaxException, IOException, InterruptedException {
    Configuration configuration = new Configuration();
    // 两个副本
    configuration.set("dfs.replication", "2");

    fileSystem = FileSystem.get(new URI("hdfs://hadoop02:9000"), configuration, "root");
}

@After
public void closeFileSystem() throws IOException {
    if (null != fileSystem) {
        fileSystem.close();
    }
}
```


# HDFS API 操作

## 文件上传

```java
@Test
public void testCopyFromLocalFile() throws IOException {
    // 上传文件，参数1：待上传文件位置，参数2：HDFS 路径
    fileSystem.copyFromLocalFile(new Path("d:\\log\\error.log"), new Path("/error.log"));
    System.out.println("上传完成");
}
```

![文件上传](/images/hadoop/client/copy-from-local.png)


将 `$HADOOP_HOME$/etc/hadoop/hdfs-site.xml` 文件，拷贝到 Java 程序的 `/resources` 文件夹下，并修改为：
```xml
<configuration>
    <!-- 副本数量改为 1 -->
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
</configuration>
```

注释掉程序中设置副本数的代码，再次上传文件，查看上传后文件副本数

结论：
1、如果程序中未设置副本数，且不存在 hdfs-site.xml 文件，则以 Hadoop 中设置的 hdfs-site.xml 中的副本数优先
2、如果程序中未设置副本数，存在 hdfs-site.xml 文件，以程序中的 hdfs-site.xml 中的副本数优先
3、如果程序中设置了副本数，且存在 hdfs-site.xml，以程序中设置的副本数优先
4、如果程序中设置了副本数，且不存在 hdfs-site.xml，以程序中设置的副本数优先

即：`Java 程序` > `resources 中的 hdfs-site.xml` > `hadoop 中的 hdfs-site.xml` > `hadoop 默认的副本数`

## 文件下载

```java
@Test
public void testCopyToLocalFile() throws IOException {
    // 从 HDFS 拷贝的本机
    fileSystem.copyToLocalFile(new Path("/error.log"), new Path("d:\\log\\copy-to-local.log"));
    // 参数1：是否删除源数据，参数2：HDFS，参数3：本地路径，参数4：是否开启本地模式校验
    // 参数4： 为true 时，下载成功后不会生成 .crc 文件，为 false 时会生成 .crc 文件
    // .crc 文件：校验数据可靠性的文件
    fileSystem.copyToLocalFile(false, new Path("/error.log"), new Path("d:\\log\\copy-to-local-1.log"), true);
}
```

## 文件删除

```java
@Test
public void testDelete() throws IOException {
    // 参数1：HDFS
    // 参数2：是否递归删除，当参数1是文件夹时，需要设置为 true，否则报错
    fileSystem.delete(new Path("/error1.log"), false);
}
```

## 文件改名

```java
@Test
public void testRename() throws IOException {
    // 参数1：要修改的 HDFS
    // 参数2：修改为 HDFS
    fileSystem.rename(new Path("/error.log"), new Path("/log.out"));
}
```

## 查看文件详情

可以查看文件名称、权限、长度、块信息等

```java
public void testListFiles() throws IOException {
    // 参数1：HDFS
    // 参数2：是否遍历
    // 返回值：获取到的文件信息迭代器
    RemoteIterator<LocatedFileStatus> files = fileSystem.listFiles(new Path("/"), false);
    while (files.hasNext()) {
        // 文件信息
        LocatedFileStatus next = files.next();
        System.out.println(next);
        for (BlockLocation blockLocation : next.getBlockLocations()) {
            // 块信息
            System.out.println(blockLocation);
        }
    }
}
```

返回值：
```json
// 文件信息
{
    "path": "hdfs://hadoop02:9000/error1.log",   // 文件路径
    "isDirectory": false,                       // 是否是文件夹
    "length":1349614,               // 文件长度
    "replication": 2,           // 副本数
    "blocksize": 134217728,     // 块大小
    "modification_time": 1574819199782, // 修改时间
    "access_time": 1574819199617,  
    "owner": "root",                // 所有者
    "group": "supergroup",          // 所有组
    "permission": "rw-r--r--",      // 权限
    "isSymlink": false
}

// 块信息
// 分别代表：起始位置，结束位置，块所在的hadoop服务器
0,1349614,hadoop03,hadoop02
```

## 判断是否是文件夹

```java
@Test
public void testIsDir() throws IOException {
    RemoteIterator<LocatedFileStatus> files = fileSystem.listFiles(new Path("/"), true);
    while (files.hasNext()) {
        System.out.println(files.next().getPath().getName() + " 是否是文件夹？" + !files.next().isFile());
    }
}
```

---

# HDFS I/O 流操作

## 文件上传

```java
@Test
public void testPutFileToHdfs() throws IOException {

    // 2、获取输入流

    FileInputStream inputStream = new FileInputStream(new File("d:/log/error.log"));

    // 3、获取输出流
    FSDataOutputStream fsDataOutputStream = fileSystem.create(new Path("/test-io.log"));

    // 4、流的对拷
    IOUtils.copyBytes(inputStream, fsDataOutputStream, configuration);

    // 5、关闭资源
    IOUtils.closeStream(inputStream);
    IOUtils.closeStream(fsDataOutputStream);

}
```

## 文件下载

```java
@Test
public void testGetFileFromHdfs() throws IOException {
    // 获取输入流
    FSDataInputStream inputStream = fileSystem.open(new Path("/log.out"));

    // 获取输出流
    FileOutputStream outputStream = new FileOutputStream(new File("d:/log/log1.out"));

    // 流的对拷
    IOUtils.copyBytes(inputStream, outputStream, configuration);

    // 关闭资源
    IOUtils.closeStream(outputStream);
    IOUtils.closeStream(inputStream);
}
```

## HDFS 文件定位读取

先往 HDFS 中上传一个大于 128M 的文件，在管理器中查看一下文件的`分块大于1`。

![大于1块的文件](/images/hadoop/client/more-block.png)

可见当前文件分为了两块存储。如果此时进行下载，会将两块数据合并起来下载。但如果只想要下载其中的一部分，现在的下载方法无法实现。

```java
// 只读取第一块的数据
public void testReadFileSeek1() throws IOException {
    // 获取输入流
    FSDataInputStream inputStream = fileSystem.open(new Path("/hadoop-2.7.2.tar.gz"));

    FileOutputStream outputStream = new FileOutputStream("d:/log/hadoop.part1");

    //  只拷贝 128 M
    byte[] buffer = new byte[1024];
    for (int i = 0; i < 1024 * 128; i++) {
        inputStream.read(buffer);
        outputStream.write(buffer);
    }

    IOUtils.closeStream(outputStream);
    IOUtils.closeStream(inputStream);

}

// 再读取第二块
public void testReadFileSeek2() throws IOException{
    // 获取输入流
    FSDataInputStream inputStream = fileSystem.open(new Path("/hadoop-2.7.2.tar.gz"));
    // 指定读取开始点
    inputStream.seek(1024 * 1024 * 128);
    // 获取输出流
    FileOutputStream outputStream = new FileOutputStream("d:/log/hadoop.part2");
    // 对拷
    IOUtils.copyBytes(inputStream, outputStream, configuration);

    // 关闭资源
    IOUtils.closeStream(outputStream);
    IOUtils.closeStream(inputStream);
}

// 拼接两块数据
// 由于当前是 windows 环境，使用 cmd 窗口拼接。
// cmd 进入两块所在目录
// 命令：type hadoop.part2 >> hadoop.part1
// 再把 part1 的后缀改为 tar.gz 即可查看
```

--

# HDFS 读写数据流程

## HDFS 写数据流程

1. 使用 FileSystem.get 创建一个`分布式文件系统`客户端，向 NameNode 请求上传文件
2. NameNode 检查 HDFS 中是否有待上传的文件（根据路径、文件名判断），如果存在该文件，则报错`文件已存在`
3. 如果 NameNode 检查后，HDFS 没有待上传的文件，则开始响应上传文件
4. 请求上传第一个 block（根据配置不同，block 大小也不同），此时会向 DataNode 请求，由 DataNode 决定可以上传到哪几个节点上
5. DataNode 返回可以上传的节点（判断条件：节点距离，负载）
6. FileSystem 创建输出流（FsDataOutputStream），与 DataNode 建立通道（串行）
7. DataNode 应答，所有可上传节点应答成功后，开始传输数据
8. 所有数据传输完成后，通知 NameNode

![hdfs 写数据流程](/images/hadoop/client/write-data.png)


### 节点距离计算

`节点距离：两个节点到最近的共同祖先的距离总和`

HDFS 写数据过程中，NameNode 会选择距离上传数据最近距离的 DataNode 接收数据，此时需要计算节点距离。

![节点距离](/images/hadoop/client/distance.png)

> (d1-r1-n0, d1-r1-n0)，由于这两个节点在同一个服务器上，此时距离为 0。即：同一节点上的进程距离为 0。
> (d1-r1-n1, d1-r1-n2)，由于这两个节点都在同一个机架上，所以 n1、n2 的共同祖先都为 r1，此时距离为 1+1=2
> (d1-r2-n0, d1-r3-n2)，这两个节点在同一个集群的不同机架上，即这两个节点的共同祖先为 d1，节点到集群还需要经过机架，所以这两个节点到共同祖先的距离都为 2，则节点距离为 2+2=4
> (d1-r2-n1, d2-r4-n1)，这两个节点也不在同一个集群，则共同祖先为最外围的“网段”，此时每个节点到“网段”的距离都为 3，所以节点距离为 3+3=6

### 机架感知（副本存储节点选择）

默认情况下，当副本数为 3 时，HDFS 的副本策略是在 `本地机架` 上的一个节点放置一个副本，在 `本地机架的另外一个节点` 上放置一个副本，最后再 `另外一个机架` 的不同节点上防止最后一个副本。

老版本的 hadoop 正好相反，是在 `本地机架` 上放置一个副本，在 `另外一个机架` 上放置 2 个副本。

![默认情况下副本选择情况](/images/hadoop/client/fuben.png)

## HDFS 读数据流程

1. 客户端请求下载文件（向 NameNode 发送请求）
2. NameNode 返回目标文件元数据
3. 客户端创建输入流
4. 客户端请求读取数据（根据距离决定从那个 DataNode 获取数据）
5. DataNode 传输数据

![读数据流程](/images/hadoop/client/read-data.png)