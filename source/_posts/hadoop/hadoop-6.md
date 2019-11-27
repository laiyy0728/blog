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