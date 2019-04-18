---
title: Activiti 工作流引擎（5）  <br /> 流程定义
date: 2018-12-07 10:06:01
updated: 2018-12-07 10:06:01
categories:
    activiti
tags:
    - Activiti
---

在上两篇博文中，已经介绍了如何创建、启动、完成一个流程，以及在流程运转过程中的一些注意点和需要用到的表结构的分析。那么，一个流程定义该如何管理？比如流程删除、流程中变量的使用、指定任务处理人等操作该如何操作？

<!-- more -->

**流程定义管理涉及对象、表**

流程定义中涉及到的对象主要是 `ProcessEngine`、`RepositoryService`、`Deployment`、`ProcessDefinition`，即：`Activiti` 核心类和仓库服务。
`ProcessDefinition`：解析 .bpmn 文件后得到的流程定义规则信息，工作流就是按照流程定义的规则执行的。
`Deployment`：部署对象，一次部署多个文件的信息。对于不需要的流程可以删除和修改。
涉及到的表结构主要有：
- act_re_deployment：部署对象表
- act_re_procdef：流程定义表
- act_ge_bytearray：资源文件表
- act_ge_property：主键生成策略表


# 查询流程、资源

## 新建一个采购审批流程，如图：

![采购审批流程](/images/activiti/bpmn.png)

## 以 Zip 形式上传并自解压

```java
InputStream inputStream = getClass().getClassLoader().getResourceAsStream("BuyBill.zip");
RepositoryService repositoryService = processEngine.getRepositoryService();
Deployment deploy = repositoryService.createDeployment()
        .name("采购流程")
        // 以 zip 形式上传 bpmn 文件
        .addZipInputStream(new ZipInputStream(inputStream))
        .deploy();
System.out.println(deploy.getId() + " --> " + deploy.getName());
```

## 查看流程定义信息

```java
// 查看流程定义
ProcessDefinitionQuery query = processEngine.getRepositoryService().createProcessDefinitionQuery();
// 查询（类比 SQL 的 where 条件）
// 流程定义的id，myProcess_1:2:22503，组成方式： key + 版本 + 自动生成的id
//        query = query.processDefinitionId("myProcess_1:2:22503");
// 流程定义的 key，有 bpmn 文件的 key 决定
query.processDefinitionKey("myProcess_1");
// 流程定义名称
//        query.processDefinitionName("");
// 流程定义版本
//        query.processDefinitionVersion(1);
// 最新版本
//        query.latestVersion();
// 版本降序排序
List<ProcessDefinition> list = query.orderByProcessDefinitionVersion().desc()
        // 总数
//                .count()
        // 列表
        .list();

if (!CollectionUtils.isEmpty(list)){
    list.forEach( temp -> System.out.println("流程定义id：" + temp.getId() + " ---> 流程定义key：" + temp.getKey() + " ---> 流程版本：" + temp.getVersion() + " 部署id：" + temp.getDeploymentId() + " 流程定义名称：" + temp.getName()));
}
```

## 查询资源文件并拷贝到本地
```java
// 通过部署资源的 deployment id 获取资源
String resourceName = "";
String deploymentId = "22501";
List<String> resourceNames = processEngine.getRepositoryService().getDeploymentResourceNames(deploymentId);
if (!CollectionUtils.isEmpty(resourceNames)) {
    resourceName = resourceNames.get(0);
    // 读取资源，根据 deploy id 和 资源名称
    InputStream inputStream = processEngine.getRepositoryService().getResourceAsStream(deploymentId, resourceName);
    // 拷贝到本地
    File file = new File("d:/" + resourceName);
    FileUtils.copyInputStreamToFile(inputStream, file);
}
```

--- 

# 删除流程定义
<br/>
> 通过部署 id 删除流程定义

```java
String deployId = "2501";
processEngine.getRepositoryService().deleteDeployment(deployId);
```

操作成功后，`act_re_deploy`、`act_ge_bytearray`、`act_re_procdef` 表中会级联删除相关数据。但是 `act_hi_*` 中保存的是历史记录，历史记录不会删除。

# 流程实例、任务的执行

主要核心类：`Execution`、`ProcessInstance`、`Task`

> Execution：执行对象，按照流程定义的规则执行一次的过程。

对应的表：
- act_ru_exection：正在执行的信息
- act_hi_procinst：已经执行完的历史流程实例信息
- act_hi_actinst：存放历史所有完成的活动

> ProcessInstance：流程实例，特质流程从开始导结束的最大执行分支。一个执行的流程中，流程实例只有一个。

需要注意：
- 如果是单例流程，执行对象 id 就是流程实例 id
- 如果一个流程有分值和聚合，那么执行对象 id 和流程实例 id 就不相同
- 一个流程中，流程实例只有一个，执行对象可以存在多个

> Task 任务：执行到某任何环节时生成的任务信息。

对应的表：
- act_ru_task：正在执行的任务
- act_hi_taskinst：已经执行完的历史任务信息

## 理解流程实例和执行对象

流程实例和执行对象是两个不通的概念，可以粗略理解为，执行对象是由流程实例创建的。在一个流程中，流程的实例只能有一个，即：一个流程就是一个实例。但是执行对象可以有多个，即：一个流程可以委派给多个执行对象去执行这个流程。

举个例子：
一个人执行*跑步* —> *打球* —> *回家* 流程，如果此时在 *跑步* 完成之后，有一个快递需要拿，拿完直接回家的话，就是一个 *跑步* —> *拿快递* —> *回家* 流程。
在这个流程里面，此人只能执行其中一个流程，但是 *打球*、*拿快递* 都在主流程：*跑步* —>... —> *回家* 中。但是如果在 *拿快递* 这个流程执行的时候，把这个流程交给了一个朋友去执行，自己还可以继续去打球的。
在这个例子里面，*跑步* —> *打球*(*拿快递*) —> *回家* 是一个流程实例。 执行*打球*、*拿快递* 的是执行对象。 即：一个流程只能有一个实例，但是可以交给多个对象去执行。

![一个流程实例，多个执行对象](/images/activiti/process_instance.png)


