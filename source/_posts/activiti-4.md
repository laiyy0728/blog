---
title: Activiti 工作流引擎（4）  <br /> 启动流程
date: 2018-12-06 11:25:11
updated: 2018-12-06 11:25:11
categories:
    Java
tags:
    - Activiti
---

ProcessEngine 的几个重要的 Service

- RepositoryService：管理流程的定义
- RuntimeService：执行管理，包括流程的启动、推进、删除流程实例等操作
- TaskService：任务管理
- HistoryService：历史管理（执行完的数据管理）
- IdentityService：可选服务，任务表单管理

<!-- more -->

# 启动流程

在上一篇末尾，已经实现了使用 RepositoryService 定义、部署一个流程。部署完成的流程需要使用 RuntimeService 来启动


## 代码示例

```java
    // 执行流程 -- 执行流程属于运行，需要获取运行时服务 RuntimeService
    RuntimeService runtimeService = processEngine.getRuntimeService();
    // startProcessInstanceById 使用 act_re_procdef 中的 ID_ 字段，该字段自动生成，不便于理解
    // startProcessInstanceByKey 使用 act_re_procdef 中的 KEY_ 字段，该字段手动执行，便于理解，但是需要格外注意不能重复
    // 如果一个流程进行了多次修改，那么 KEY_ 和 NAME_ 必须一样，且运行时执行最后一个 VERSION_ 版本
    // 取得流程实例
    ProcessInstance instance = runtimeService.startProcessInstanceByKey("levelBill");
    System.out.println("流程实例id：" + instance.getId() + " ---> 流程定义id： " + instance.getProcessDefinitionId());
```

控制台打印结果为：
```
流程实例id：5001 ---> 流程定义id： levelBill:2:2503
```

查看 act_ru_execution 

![启动任务](/images/activiti/start_1.png)

- PROC_INST_ID_：对应 act_ge_property 中的 next.dbid 的值
- PARENT_ID_：对应上一步的 act_ru_execution 的 ID_
- ROOT_PROC_INST_ID_：对应运行实例的 id
- ACT_ID_：对应当前流程进行到哪一步了
- PROC_DEF_ID_：对应流程定义的 id(act_re_procdef 中的 ID_)


---

# 任务查询

在第一步启动任务后，任务流程自动进入第一步中。在本例中，相当于 请假流程已经开始，需要 zhangsan 进行审批。那么，就需要获取 zhangsan 的任务列表。

获取任务列表，顾名思义就需要`任务服务`，通过任务服务查询出任务列表

## 代码示例
```java
    // 任务办理人（第一步是zhangsan办理）
    String user = "zhangsan";
    // 获取任务服务
    TaskService taskService = processEngine.getTaskService();
    // 创建任务查询对象
    TaskQuery taskQuery = taskService.createTaskQuery();
    // 指定办理人，获取办理人的任务列表
    List<Task> list = taskQuery.taskAssignee(user).list();
    // 连理任务列表
    if (!CollectionUtils.isEmpty(list)){
        for (Task task : list) {
            System.out.println("任务办理人：" + task.getAssignee() + " --> 任务id：" + task.getId() + " --> 任务名称：" + task.getName());
        }
    }
```

在控制台中可以看到，需要张三办理的任务完整打印：
```
任务办理人：zhangsan --> 任务id：5005 --> 任务名称：first
任务办理人：zhangsan --> 任务id：7505 --> 任务名称：first
```

## 对照数据库

查询 act_ru_task 表，可以看到当前正在执行的流程数据

![正在运行的任务](/images/activiti/task.png)

- EXECUTION_ID：流程实例的id，对应 act_ru_exection 的 ID_
- PROC_DEF_ID_：流程定义的id，对应 act_re_procdef 的 ID_
- TASK_DEF_KEY_：当前流程 id，对应 act_ru_exection 的 ACT_ID_
- ASSIGNEE_：执行人，对应 BPMN 文件中指定的当前流程执行人
- NAME_：任务名称，对应 BPMN 文件中指定的当前流程的名称


---

# 任务完成

完成一个任务，即结束当前步骤，进入下一个步骤（并不是完成整改流程）

## 代码示例
```java
    // 完成这个任务，即：当前步骤完成，进行下一个步骤
    String taskId = "5005"; // 指定需要完成的任务 id
    // 完成任务，也需要任务服务
    TaskService taskService = processEngine.getTaskService();
    // 完成任务
    taskService.complete(taskId);
```

## 对照数据库

查询 act_ru_task 表

![正在运行的任务](/images/activiti/task_1.png)

与上面任务执行后获取到的表进行对比，可以看到，`ID_`、`NAME_`、`TASK_DEK_KEY_`、`ASSIGENEE_` 均已变更为当前步骤的数据，而上一步的数据已经自动删除。

> 自动删除执行过的步骤，可以保证正在运行的任务表足够小，无冗余，可以更大程度的保证流程处理的效率

## 任务结束

如果修改 taskId，运行任务完成操作，在最后一步以后，会自动完成流程。完成后的流程，在 `act_ru_task`、`act_ru_execution` 是没有记录的，因为这些流程已经完成，不是进行中的状态。这样做可以保证运行中的任务表足够小，效率足够快。

任务结束后，历史流程可以在  act_hi_* 中查看到。