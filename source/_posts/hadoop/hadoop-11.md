---
title: hadoop（11） Map Reduce <BR /> MapReduce 框架原理
date: 2019-12-04 16:17:08
updated: 2019-12-04 16:17:08
categories:
    [hadoop, map-reduce]
tags:
    [hadoop, map-reduce]
---

在了解了 Hadoop 的序列化操作，实现了基本的 Bean 序列化的一个 demo，接下来分析一下 MapReduce 的框架原理。

<!-- more -->

# InputFormat

## 切片与MapTask 并行度决定机制

MapTask 的并行度决定 Map 阶段的任务处理并发度，进而影响整个 Job 的处理速度。

问题：
> 一个 1G 的数据，启动 8 个MapTask，可以提高集群的并发处理能力。但是如果是一个 1K 的数据，也启动 8 个MapTask，会提高性能吗？
> MapTask 是否越多越好？
> 什么因素会影响到 MapTask 的并行度？

### MapTask并行度决定机制

前置概念：
> 数据块：Block 在 HDFS 物理上把数据分成一块一块的。
> 数据切片：在逻辑上对输入进行分片，并不会在磁盘上将其分片存储。

现在，假设有一个 300M 的数据，分别存放在 DataNode 1、2、3 上，DataNode1 上存储 0~128M 数据，DataNode2 上存储 128~256M 数据，DataNode3 上存储 256~300M 数据。
如果数据切片大小为 100M，则读取第一个切片没有问题，当读取第2、3个切片时，需要将DataNode1 上的剩余的数据拷贝到 MapTask2 上，将 DataNode2 上剩余的数据拷贝到 MapTask3 上，这样会存在大量的数据 IO，严重影响性能。

![切片大小为 100M](/images/hadoop/map-reduce/100-split.png)

如果数据切片大小为 128M（即与 Block 大小一致），此时，每个 MapTask 都读取 128M 数据，则可以分别运行在三台 DataNode 上，没有数据拷贝，此时性能最高。


> MapTask 并行度决定机制

1. 一个 Job 的 Map 阶段并行度由客户端在提交 Job 时的切片数决定
2. 每个切片分配一个 MapTask 并行实例处理
3. 默认情况下，切片大小等于 BlockSize
4. 切片时不考虑数据集整体，而是逐个针对每个文件单独切片

### Job 提交流程、切片源码

在 Job 调用 `job.waitForCompletion` 时，进行任务提交。此方法会调用 `submit()` 方法进行真正的提交。

```java
public boolean waitForCompletion(boolean verbose
                                ) throws IOException, InterruptedException,
                                        ClassNotFoundException {
    if (state == JobState.DEFINE) {
        // 调用真正的提交
        submit();
    }
    if (verbose) {
        // 打印日志
        monitorAndPrintJob();
    } else {
        // 忽略
    }
    return isSuccessful();
}
```

```java
public void submit() 
        throws IOException, InterruptedException, ClassNotFoundException {
    // 判断任务状态
    ensureState(JobState.DEFINE);
    // 将老旧的 API 转换为新的 API，为兼容性考虑
    setUseNewAPI();
    // 连接
    connect();
    final JobSubmitter submitter = 
        getJobSubmitter(cluster.getFileSystem(), cluster.getClient());
    status = ugi.doAs(new PrivilegedExceptionAction<JobStatus>() {
        public JobStatus run() throws IOException, InterruptedException, 
        ClassNotFoundException {
            // 提交任务的详细信息
            return submitter.submitJobInternal(Job.this, cluster);
        }
    });
    state = JobState.RUNNING;
    LOG.info("The url to track the job: " + getTrackingURL());
}
```

> connect 详细操作流程

```java
private synchronized void connect()
          throws IOException, InterruptedException, ClassNotFoundException {
    if (cluster == null) {
      cluster = 
        ugi.doAs(new PrivilegedExceptionAction<Cluster>() {
                   public Cluster run()
                          throws IOException, InterruptedException, 
                                 ClassNotFoundException {
                     // 创建一个新的 Cluster
                     return new Cluster(getConfiguration());
                   }
                 });
    }
  }
```

```java
public Cluster(Configuration conf) throws IOException {
    this(null, conf);
}

public Cluster(InetSocketAddress jobTrackAddr, Configuration conf) 
    throws IOException {
    this.conf = conf;
    this.ugi = UserGroupInformation.getCurrentUser();
    // Cluster 初始化
    initialize(jobTrackAddr, conf);
}
```

```java
private void initialize(InetSocketAddress jobTrackAddr, Configuration conf)
      throws IOException {

    synchronized (frameworkLoader) {
        for (ClientProtocolProvider provider : frameworkLoader) {
        ClientProtocol clientProtocol = null; 
        try {
            if (jobTrackAddr == null) {
                // 创建一个 LocalJobRunner（在本地运行时）
                clientProtocol = provider.create(conf);
            } else {
                // 创建一个 YARNRunner（在集群运行时）
                clientProtocol = provider.create(jobTrackAddr, conf);
            }
            // 省略
        }
    }

    // 省略
}
```

> connect 连接成功，提交任务详细信息

```java
JobStatus submitJobInternal(Job job, Cluster cluster) 
throws ClassNotFoundException, InterruptedException, IOException {

    // 校验输出路径
    checkSpecs(job);

    Configuration conf = job.getConfiguration();
    // 缓存处理
    addMRFrameworkToDistributedCache(conf);

    // 任务临时目录， tmp/hadoop-Administrator\mapred\staging，每次运行任务都会在这个目录下创建一个文件夹，将所需数据都保存在内
    // 当任务执行结束后，会删除文件夹内的所有数据
    Path jobStagingArea = JobSubmissionFiles.getStagingDir(cluster, conf);
    
    // 获取网络 ip
    InetAddress ip = InetAddress.getLocalHost();
    // 省略

    // 生成一个唯一的 jobId
    JobID jobId = submitClient.getNewJobID();
    job.setJobID(jobId);
    Path submitJobDir = new Path(jobStagingArea, jobId.toString());
    JobStatus status = null;
    try {
        // 省略

        // 提交文件的信息到之前创建的文件夹下（本机下不会提交）
        copyAndConfigureFiles(job, submitJobDir);

        Path submitJobFile = JobSubmissionFiles.getJobConfPath(submitJobDir);
        
        // 写入切片信息到文件夹
        int maps = writeSplits(job, submitJobDir);
        conf.setInt(MRJobConfig.NUM_MAPS, maps);

        // 省略

        // 写入任务信息到文件夹
        writeConf(conf, submitJobFile);
        
        printTokens(jobId, job.getCredentials());

        // 提交完成后，删除文件夹内信息
        status = submitClient.submitJob(
            jobId, submitJobDir.toString(), job.getCredentials());

        // 省略
    } finally {
        // 省略
    }
}
```

![Hadoop 任务临时路径](/images/hadoop/map-reduce/job-staging.png)
![hadoop 临时切片文件](/images/hadoop/map-reduce/split-file.png)
![hadoop Job 提交流程](/images/hadoop/map-reduce/job-submit.png)