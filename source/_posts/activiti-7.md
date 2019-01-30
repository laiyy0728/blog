---
title: Activiti 工作流引擎（7）  <br /> 流程分支、排他网关、动态处理人
date: 2018-12-17 09:52:28
updated: 2018-12-17 09:52:28
categories:
    Java
tags:
    - Activiti
---

除了一个流程进行到底的工作流之外，还有一些有分支的工作流。如：下属在申报审批一条信息的时候，如果这条信息不算太重要，可以由经理审批；如果这条信息重要，需要由老板进行审批。

<!-- more -->

![分支流程](/images/activiti/sequence.png)

这时，就需要有流程分支。流程分支的作用就是处理一个流程中存在多个子分支时，根据不同的子分支条件，进行不同的子分支处理。

--- 

# 在流程图连线中确定条件

依照上图做出一个有分支的流程，在这个流程中，处理当天信息有 2 个分支，一个是 *不重要* 的信息交给`经理`处理，而 *重要* 的分支交给`老板`处理。无论谁处理过，这个流程都结束。

## 流程图中确定条件

点击分支流程：*处理当天信息* —> *经理(老板)* 的`连线`， 在`Name`中填入备注信息，在`Condition`中填入条件信息。需要注意：
> 1、`Condition`必须为`布尔`类型；
> 2、判断条件必须作为参数在执行中传入；
> 3、尽量不使用中文

![分支条件](/images/activiti/condition.png)

`Condition`：满足这个条件的时候，走这一步流程。即：满足 `${message=='no'}` 走*经理*流程；满足`${message=='yes'}`走*老板*流程。

## 代码示例

流程部署、启动不再示例。直接从流程分支开始。
```java
String taskId = "85005";
Map<String, Object> params = new HashMap<>();
params.put("message", "yes");
processEngine.getTaskService().complete(taskId, params);
System.out.println("执行“老板”分支，执行完毕");
```

在执行任务时，直接传入 `message`，需要保证 `message` 与 `Condition` 一致。
如果在传入`message`时，指定的值与`Contidition`不一致，在本例中即`message`传入值不是 yes/no，这时在执行时，由于判断条件错误，会导致后台报错，但流程不会进行。

错误信息：
```java
org.activiti.engine.ActivitiException: No outgoing sequence flow of element '_4' could be selected for continuing the process
	at org.activiti.engine.impl.agenda.TakeOutgoingSequenceFlowsOperation.leaveFlowNode(TakeOutgoingSequenceFlowsOperation.java:172)
	at org.activiti.engine.impl.agenda.TakeOutgoingSequenceFlowsOperation.handleFlowNode(TakeOutgoingSequenceFlowsOperation.java:87)
	at org.activiti.engine.impl.agenda.TakeOutgoingSequenceFlowsOperation.run(TakeOutgoingSequenceFlowsOperation.java:75)
	at org.activiti.engine.impl.interceptor.CommandInvoker.executeOperation(CommandInvoker.java:73)
	at org.activiti.engine.impl.interceptor.CommandInvoker.executeOperations(CommandInvoker.java:57)
	at org.activiti.engine.impl.interceptor.CommandInvoker.execute(CommandInvoker.java:42)
	at org.activiti.engine.impl.interceptor.TransactionContextInterceptor.execute(TransactionContextInterceptor.java:48)
	at org.activiti.engine.impl.interceptor.CommandContextInterceptor.execute(CommandContextInterceptor.java:63)
	at org.activiti.engine.impl.interceptor.LogInterceptor.execute(LogInterceptor.java:29)
	at org.activiti.engine.impl.cfg.CommandExecutorImpl.execute(CommandExecutorImpl.java:44)
	at org.activiti.engine.impl.cfg.CommandExecutorImpl.execute(CommandExecutorImpl.java:39)
	at org.activiti.engine.impl.TaskServiceImpl.complete(TaskServiceImpl.java:186)
	at com.laiyy.activiti.config.SequenceFlow.sequenceFlowStart(SequenceFlow.java:40)
```
`_4` 是执行这个任务的流程id，bpmn 文件中指定的`处理当天信息`这个步骤的 id。即在执行流程分支时，由于判断条件出错，导致流程不能往下执行。

---

# 排他网关

排他网关主要用于在一个流程中，存在多个分支流程，但是无默认分支的情况。可以解决上面例子中出现的`Condition`不匹配的问题。在不能匹配`Condition`时进入默认流程分支。

如在银行办理业务时，有 *普通窗口*、*VIP窗口* 和银行的 *后台窗口*，在申请办理业务人为后台用户时，走 *后台窗口*，为 VIP 用户时，走 *VIP窗口*，其余默认走 *普通窗口*。
与上例中的 *经理/老板* 流程不同的是，在 *普通窗口* 的流程中，是没有判断条件的。这是一个简单的排他网关。

![排他网关](/images/activiti/exclusive_gateway.png)

此时按照上例中的代码继续执行时，会发现，如果 visitor 满足 3，则进入`后台窗口`，满足 2 则进入 `vip窗口`，否则都进入`普通窗口`


# 动态指定处理人

可以使用类似于`el表达式`的方式：$userId，设置一个变量，在处理过程中动态的给这个变量赋值，即可实现动态指定。

![动态指定处理人](/images/activiti/assignee.png)

## 设置处理人

在进行流程处理时，设置下一步执行人
```java
Map<String, Object> params =~ new HashMap<>();
params.put("userId", "zhangsan");
ProcessInstance processInstance = processEngine.getRuntimeService()
                    .startProcessInstanceByKey(processDefikey, params);



// 执行结束后，根据处理人查看任务信息
String user = "zhangsan";
// 获取任务服务
TaskService taskService = processEngine.getTaskService();
// 创建任务查询对象
TaskQuery taskQuery = taskService.createTaskQuery();
// 指定办理人，获取办理人的任务列表
List<Task> list = taskQuery.taskAssignee(user).list();
// 连理任务列表
if (!CollectionUtils.isEmpty(list)) {
for (Task task : list) {
    System.out.println("任务办理人：" + task.getAssignee() + " --> 任务id：" + task.getId() + " --> 任务名称：" + task.getName());
}
```
