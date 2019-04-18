---
title: Activiti 工作流引擎（6）  <br /> 历史流程、流程变量
date: 2018-12-14 11:45:06
updated: 2018-12-14 11:45:06
categories:
    activiti
tags:
    - Activiti
---

Activiti 不仅仅能执行流程、获取到当前流程的信息，也可以获取到已经执行过的流程信息、任务信息。
历史任务：流程执行的每一步都是一个任务，历史任务列表是所有流程的每一步执行情况。
<!-- more -->
需要用到的表：act_hi_taskinst、act_hi_procinst

---

# 查询历史流程

> 查看历史执行实例：

代码示例
```java
HistoryService historyService = processEngine.getHistoryService();
List<HistoricProcessInstance> list = historyService.createHistoricProcessInstanceQuery().list();
if (!CollectionUtils.isEmpty(list)) {
    list.forEach(item -> {
        System.out.println("流程实例id" + item.getId());
        System.out.println("流程实例定义id" + item.getProcessDefinitionId());
        System.out.println("流程开始时间" + item.getStartTime());
        System.out.println("流程结束时间" + item.getEndTime());
        System.out.println("=================");
    });
}
```

> 查看历史任务（每一步）

代码实例
```java
HistoryService historyService = processEngine.getHistoryService();
List<HistoricTaskInstance> list = historyService.createHistoricTaskInstanceQuery().list();
if (!CollectionUtils.isEmpty(list)){
    list.forEach(item -> {
        System.out.println("历史流程实例id：" + item.getId());
        System.out.println("历史流程定义id：" + item.getTaskDefinitionKey());
        System.out.println("任务名称：" + item.getName());
        System.out.println("处理人：" + item.getAssignee());
        System.out.println("================");
    });
}
```

---

# 流程变量

如：在支付流程中，*支付* —> *填写金额* —> *确认/取消*，在*确认/取消*时，需要查看到当前支付金额，这个金额就是在支付流程中的变量。

流程变量设计到的表：
1、act_ru_variable：正在执行的流程变量表
2、act_hi_varinst：流程变量历史表

## 设置流程变量值

可以设置普通的 POJO，但是需要实现序列化接口

### 通过 taskService 设置变量

> 通过 setVariable 设置
```java
/**
  * 通过 set 设置
  */
// taskId 任务id，范围比 runtime 小
// variableName 变量名
// value 变量值
taskService.setVariable(taskId, variableName, value);
// 设置本执行对象的变量，该对象的作用域只在当前的执行对象
taskService.setVariableLocal(taskId, variableName, value);
// 设置多个变量，values： Map<String, Object>
taskService.setVariables(taskId, values);

/**
  * 完成任务时设置
  */
taskService.complete(taskId, values [, localScope])

```

### 通过 runtimeService 设置

```java
/**
  * 通过 set 设置
  */
// executionId 执行对象id
// variableName 变量名
// value 变量值
runtimeService.setVariable(executionId, variableName, value);
// 设置本执行对象的变量，该对象的作用域只在当前的执行对象
runtimeService.setVariableLocal(executionId, variableName, value);
// 设置多个变量，values： Map<String, Object>
runtimeService.setVariables(executionId, values);

/**
  * 启动时设置
  */
// processKey 任务 key， values Map<String, Object>
runtimeService.startProcessInstanceByKey(processKey, values)
```

## 获取流程变量

### 使用 taskService 获取流程变量
```java
runtimeService.getVariable(executionId, key) 去单个变量
runtimeService.getVariableLocal(executionId, key) 取本执行对象的单个变量
runtimeService.getVariables(executionId) // 取多个变量
```

### 使用 runtimeService 获取流程变量
```java
taskService.getVariable(taskId, key)
taskService.getVariableLocal(taskId, key)
taskService.getVariables(taskId)
```

## 变量支持的类型

> 简单类型

String、boolean、Integer、double、Date 等

> 自定义对象

自定义的 POJO

