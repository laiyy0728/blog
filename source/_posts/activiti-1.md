---
title: Activiti 工作流引擎（1）  <br /> 工作流基础了解
date: 2018-12-05 11:43:06
updated: 2018-12-05 11:43:06
categories:
    Java
tags:
    - Activiti
---

# 什么是工作流

工作流（Workflow），指“业务过程的部分或整体在计算机应用环境下的自动化”。是对工作流程及其各操作步骤之间业务规则的抽象、概括描述。在计算机中，工作流属于计算机支持的协同工作（CSCW）的一部分。后者是普遍地研究一个群体如何在计算机的帮助下实现协同工作的。
工作流主要解决的主要问题是：为了实现某个业务目标，利用计算机在多个参与者之间按某种预定规则自动传递文档、信息或者任务。
<!-- more -->

## 常见的工作流的地方

> * OA 审批功能
> * 电子政务上传下达
> * 物流运输流程记录
> * ...

## 使用工作流和不使用工作流的区别

以学校请假为例，如果一个学生请假需要经过以下流程：** 填写请见条 --> 提交给老师 --> 老师审批*（通过，不通过）* --> 提交到年级处 --> 年级处审批*（通过，不通过）* --> 提交给教务处 --> 教务处审批 *（通过，不通过）* --> 提交到校长 --> 校长审批*（通过，不通过）* --> 结束 **

---
  
# Activiti 工作流

常见的开源工作流引擎框架有：OSWrokFlow、jBPM（jboss business process management）、Activiti（对 jBPM 的升级）、Spring WorkFlow 等

## Activiti 的简单认识

### ProcessEngine

ProcessEngine 是 Activiti 的核心工作类，可以由该类获取到其他服务（历史服务、仓库服务、任务服务、角色 / 参与者服务）

历史服务：之前运行过的所有流程即为历史服务
仓库服务：定义好的流程需要保存到一个仓库中（一般为数据库），该数据库中保存的流程，解析该流程的服务即为仓库服务
任务服务：定义好的流程中的每一步即为一个任务服务
角色 / 参与者服务：执行流程中步骤的人、角色，即为一个 角色 / 参与者服务

### BPMN

BPMN： 业务流程建模与标注（Business Process Model and Notation），描述流程的基本符号，包括这些土元如果组成一个业务流程图（Business Process Diagram）

以一个简单的业务流程图为例： 第一个圆圈代表流程开始，审批流程为： 提交 -> 经纪人 -> 老总，最后一个加粗的圆圈代表流程结束。每一个起始点、流程审批点、结束点，都是一个最基本的 BPMN，这些点组合在一起，整个图可以称为一个最基本的 业务流程图。

![业务流程图](/images/activiti/business_process_diagram.png)


### 配置文件

activiti.cfg.xml： Activiti 核心配置文件，配置流程引擎创建工具的基本参数和数据库连接参数

logging.properties： log4j 日志打印

---

# 数据库表

Activiti 数据库总共有 23 张表，所有表都是以 ACT_ 开头，第二部分表示表的用途，一般用两个字母标示，用于和服务的 API 对应。

> * act_ge_* ： 通用数据，用于不同场景下，如：存放资源文件
> * act_hi_* ： hi 代表 history。包含历史数据，比如：历史流程实例、变量、任务等
> * act_re_* ： re 代表 repository。这个前缀的表包含了定义流程和流程静态资源（图片、规则等）
> * act_ru_* ： ru 代表 runtime。包含了流程实例、任务、变量、异步任务等运行中的数据。Activiti 只在了流程实例执行过程中保存这些数据，在流程结束后就会删除这些记录，这样可以保证运行时表一直很小，速度很快）
> * act_id_* ： id 代表 identity。包含身份信息，比如：用户、组等

## 流程规则表: act_re_*

| 表名 | 作用 |
|:-:|:-:|
| act_re_deployment | 部署信息 |
| act_re_model | 流程设计模型部署表 |
| act_re_procdef|流程定义数据表|

## 运行时数据库表：act_ru_*
| 表名 | 作用 |
|:-:|:-:|
|act_ru_execution|运行时流程执行实例表|
|act_ru_identitylink|运行时流程人员表，主要存储任务节点与参与者的相关信息|
|act_ru_task|运行时任务节点表|
|act_ru_variable|运行时流程变量数据表|

## 历史数据库表： act_hi_*
| 表名 | 作用 |
|:-:|:-:|
|act_hi_actinst|历史节点表|
|act_hi_attachment|历史附件表|
|act_hi_comment|历史意见表|
|act_hi_identitylink|历史流程人员表|
|act_hi_detail|历史详情表，提供历史变量的查询|
|act_hi_procinst|历史流程实例表（常用）|
|act_hi_taskinst|历史任务实例表（常用）|
|act_hi_varinst|历史变量表（常用）|

## 组织结构表：act_id_*
| 表名 | 作用 |
|:-:|:-:|
|act_id_group|用户组信息|
|act_id_info|用户扩展信息|
|act_id_membership|用户与用户组队员信息|
|act_id_user|用户信息|

## 通用数据表： act_ge_*
| 表名 | 作用 |
|:-:|:-:|
| act_ge_bytearray| 二进制数据表|
| act_ge_property| 属性数据表，存储整个流程引擎级别的数据，初始化时会默认插入三条数据|


