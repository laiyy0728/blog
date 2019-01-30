---
title: Activiti 工作流引擎（2） <br />  使用 IDEA 创建工作流
date: 2018-12-06 09:30:54
updated: 2018-12-06 09:30:54
categories:
    Java
tags:
    - Activiti
---

## 安装 actiBPM 插件

在 IDEA 中选择： File --> Setting --> Plugins --> Browse repositories --> 查询 actiBPM 并下载安装
## 创建一个 BPM 文件

<!-- more -->

在 resources（静态文件存储路径）上右键新建，选择 BpmnFile 或者 BPMN File 进行创建
![创建 BPMN 文件](/images/activiti/create_bpmn.png)

## 设置流程

一个已经设置好的流程：
![BPMN 文件](/images/activiti/bmpn.png)


常用的几个控制器：
- startEvent：流程开始
- endEvent：流程结束
- UserTask：用户操作任务
- ScriptTask：脚本操作任务
- ServiceTask：业务操作任务
- MailTask：邮件任务

将右侧控制器拖拽至中间白板上，即可设置控制器属性。然后将鼠标放在控制器中央位置，会有一个圆点，选中圆点下拉至另外一个控制器，即可设置流程顺序。

### 设置整个 bpmn 文件属性

点击空白处，可在右侧看到整个 bpmn 文件属性，可以根据自己的实际需求进行设置

![BPMN 文件属性](/images/activiti/bmpn_1.png)

常用属性：
- Id：可以看做 BPMN 在整个流程中的文件唯一标识
- Name： 可以看做 BPMN 文件的别名（实际名称是创建 BPMN 文件时的名称）

### 设置开始、结束控制器

点击开始、结束控制器，可根据自己实际需求设置控制器属性
![BPMN 开始、结束控制器](/images/activiti/bmpn_2.png)


常用属性：
- Id：这个开始、结束控制器的 id，尽量不要更改
- Name：这个开始、结束控制器的名称，更改后可以在中间图标出立即显示；也可以双击图标进行更改

### 设置中间流程控制器

点击中间流程控制器，设置控制器属性
![BPMN 流程控制器](/images/activiti/bmpn_3.png)

常用属性：
- Id：流程控制器 Id，尽量不要更改
- Name： 控制器的名称，更改后可以在中间图标出立即显示；也可以双击图标进行更改
- Assignee：谁管理这个流程控制器（可以是用户、角色等）

## 设置 BPMN 需要注意的点：
在流程控制器中，Id 尽量不要更改。如果更改的话，必须要保证 id 不能重复，否则会出现同一个 id 对应两个流程控制，导致无法确定进行哪一步的流程，这样会出现错误。

---

## bpmn 文件

将 bpmn 文件以文本形式打开，可以发现，bpmn 文件实际上是一个 xml 文件。以刚才创建好的 bpmn 文件为例
```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<definitions xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL" xmlns:activiti="http://activiti.org/bpmn" xmlns:bpmndi="http://www.omg.org/spec/BPMN/20100524/DI" xmlns:dc="http://www.omg.org/spec/DD/20100524/DC" xmlns:di="http://www.omg.org/spec/DD/20100524/DI" xmlns:tns="http://www.activiti.org/testm1544000001944" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" expressionLanguage="http://www.w3.org/1999/XPath" id="m1544000001944" name="" targetNamespace="http://www.activiti.org/testm1544000001944" typeLanguage="http://www.w3.org/2001/XMLSchema">
    <!--  id 即为刚才设置的 “整个 BPMN 文件” 的 id， name 为 Name  -->
    <process id="levelBill" isClosed="false" isExecutable="true" name="LevelBill" processType="None">
        <!-- 开始节点， id 为 _2， name 为 start -->
        <startEvent id="_2" name="start"/>
        <!-- 用户操作节点，id 为 _3，name 为 first，操作人为 zhangsan -->
        <userTask activiti:assignee="zhangsan" activiti:exclusive="true" id="_3" name="first"/>
        <userTask activiti:assignee="lsii" activiti:exclusive="true" id="_5" name="second"/>
        <userTask activiti:assignee="wangwu" activiti:exclusive="true" id="_7" name="third"/>
        <!-- 结束节点，id为 _9，name 为 end -->
        <endEvent id="_9" name="end"/>

        <!-- 流程控制（即两个控制器之间的箭头连线），从 _2 步骤指向 _3 步骤，即第一步为 _2：start，第二步为 _3：first -->
        <sequenceFlow id="_4" sourceRef="_2" targetRef="_3"/>
        <sequenceFlow id="_6" sourceRef="_3" targetRef="_5"/>
        <sequenceFlow id="_8" sourceRef="_5" targetRef="_7"/>
        <!-- 流程控制，从 _7 步骤执行 _9，即最后一步为 从 _7：third 向 _9：end 流通 -->
        <sequenceFlow id="_10" sourceRef="_7" targetRef="_9"/>
    </process>

    <!-- 下方为 bpmn 各节点样式、坐标等，通过拖拽自动生成，可改文件微调  -->
    <bpmndi:BPMNDiagram documentation="background=#3C3F41;count=1;horizontalcount=1;orientation=0;width=842.4;height=1195.2;imageableWidth=832.4;imageableHeight=1185.2;imageableX=5.0;imageableY=5.0" id="Diagram-_1" name="New Diagram">
        <bpmndi:BPMNPlane bpmnElement="levelBill">
        <!-- 每个节点的位置、宽高等调整，bpmnElement 指向节点 id -->
        <bpmndi:BPMNShape bpmnElement="_2" id="Shape-_2">
            <dc:Bounds height="32.0" width="32.0" x="450.0" y="70.0"/>
            <bpmndi:BPMNLabel>
            <dc:Bounds height="32.0" width="32.0" x="0.0" y="0.0"/>
            </bpmndi:BPMNLabel>
        </bpmndi:BPMNShape>
        <bpmndi:BPMNShape bpmnElement="_3" id="Shape-_3">
            <dc:Bounds height="55.0" width="85.0" x="425.0" y="160.0"/>
            <bpmndi:BPMNLabel>
            <dc:Bounds height="55.0" width="85.0" x="0.0" y="0.0"/>
            </bpmndi:BPMNLabel>
        </bpmndi:BPMNShape>
        <bpmndi:BPMNShape bpmnElement="_5" id="Shape-_5">
            <dc:Bounds height="55.0" width="85.0" x="425.0" y="260.0"/>
            <bpmndi:BPMNLabel>
            <dc:Bounds height="55.0" width="85.0" x="0.0" y="0.0"/>
            </bpmndi:BPMNLabel>
        </bpmndi:BPMNShape>
        <bpmndi:BPMNShape bpmnElement="_7" id="Shape-_7">
            <dc:Bounds height="55.0" width="85.0" x="425.0" y="370.0"/>
            <bpmndi:BPMNLabel>
            <dc:Bounds height="55.0" width="85.0" x="0.0" y="0.0"/>
            </bpmndi:BPMNLabel>
        </bpmndi:BPMNShape>
        <bpmndi:BPMNShape bpmnElement="_9" id="Shape-_9">
            <dc:Bounds height="32.0" width="32.0" x="455.0" y="465.0"/>
            <bpmndi:BPMNLabel>
            <dc:Bounds height="32.0" width="32.0" x="0.0" y="0.0"/>
            </bpmndi:BPMNLabel>
        </bpmndi:BPMNShape>
        <!-- 两个节点之间的链接，bpmnElement 指向流程链接 id -->
        <bpmndi:BPMNEdge bpmnElement="_4" id="BPMNEdge__4" sourceElement="_2" targetElement="_3">
            <di:waypoint x="466.0" y="102.0"/>
            <di:waypoint x="466.0" y="160.0"/>
            <bpmndi:BPMNLabel>
            <dc:Bounds height="0.0" width="0.0" x="0.0" y="0.0"/>
            </bpmndi:BPMNLabel>
        </bpmndi:BPMNEdge>
        <bpmndi:BPMNEdge bpmnElement="_6" id="BPMNEdge__6" sourceElement="_3" targetElement="_5">
            <di:waypoint x="467.5" y="215.0"/>
            <di:waypoint x="467.5" y="260.0"/>
            <bpmndi:BPMNLabel>
            <dc:Bounds height="0.0" width="0.0" x="0.0" y="0.0"/>
            </bpmndi:BPMNLabel>
        </bpmndi:BPMNEdge>
        <bpmndi:BPMNEdge bpmnElement="_8" id="BPMNEdge__8" sourceElement="_5" targetElement="_7">
            <di:waypoint x="467.5" y="315.0"/>
            <di:waypoint x="467.5" y="370.0"/>
            <bpmndi:BPMNLabel>
            <dc:Bounds height="0.0" width="0.0" x="0.0" y="0.0"/>
            </bpmndi:BPMNLabel>
        </bpmndi:BPMNEdge>
        <bpmndi:BPMNEdge bpmnElement="_10" id="BPMNEdge__10" sourceElement="_7" targetElement="_9">
            <di:waypoint x="471.0" y="425.0"/>
            <di:waypoint x="471.0" y="465.0"/>
            <bpmndi:BPMNLabel>
            <dc:Bounds height="0.0" width="0.0" x="0.0" y="0.0"/>
            </bpmndi:BPMNLabel>
        </bpmndi:BPMNEdge>
        </bpmndi:BPMNPlane>
    </bpmndi:BPMNDiagram>
</definitions>
```


