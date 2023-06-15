---
layout: post
title: 'Camunda源码剖析之通过API生成BPMN文件(七)' 
date: 2023-06-10
author: 李新
tags:  Camunda
---

### (1). 概述
最近有这样一个需求,要实现一个编排系统,画布产生的数据是json,自己懒得去写一套工作流,就想着把json向工作流(Camunda/Activiti/Flowable)靠拢,实际就是把json转换成xml. 

### (2). 通过Camunda Modeler绘制流程图
!["Camunda example"](/assets/camunda/imgs/camunda-example.png) 

### (3). 通过Camunda Modeler绘制的原始xml
```
<?xml version="1.0" encoding="UTF-8"?>
<bpmn:definitions xmlns:bpmn="http://www.omg.org/spec/BPMN/20100524/MODEL" 
                  xmlns:bpmndi="http://www.omg.org/spec/BPMN/20100524/DI" 
                  xmlns:dc="http://www.omg.org/spec/DD/20100524/DC" 
                  xmlns:di="http://www.omg.org/spec/DD/20100524/DI" 
                  xmlns:modeler="http://camunda.org/schema/modeler/1.0" 
                  id="Definitions_016seq7" 
                  targetNamespace="http://bpmn.io/schema/bpmn" 
                  exporter="Camunda Modeler" 
                  exporterVersion="5.5.0-nightly.20221024" 
                  modeler:executionPlatform="Camunda Platform" 
                  modeler:executionPlatformVersion="7.18.0">
  <bpmn:process id="Process_19ixb1h" isExecutable="true">
    <bpmn:startEvent id="star" name="开始">
      <bpmn:outgoing>Flow_13jlccj</bpmn:outgoing>
    </bpmn:startEvent>
    <bpmn:task id="userTask" name="Task1">
      <bpmn:incoming>Flow_13jlccj</bpmn:incoming>
      <bpmn:outgoing>Flow_0k8cen8</bpmn:outgoing>
    </bpmn:task>
    <bpmn:sequenceFlow id="Flow_13jlccj" sourceRef="star" targetRef="userTask" />
    <bpmn:endEvent id="end" name="结束">
      <bpmn:incoming>Flow_0k8cen8</bpmn:incoming>
    </bpmn:endEvent>
    <bpmn:sequenceFlow id="Flow_0k8cen8" sourceRef="userTask" targetRef="end" />
  </bpmn:process>
  <bpmndi:BPMNDiagram id="BPMNDiagram_1">
    <bpmndi:BPMNPlane id="BPMNPlane_1" bpmnElement="Process_19ixb1h">
      <bpmndi:BPMNShape id="_BPMNShape_StartEvent_2" bpmnElement="star">
        <dc:Bounds x="179" y="99" width="36" height="36" />
        <bpmndi:BPMNLabel>
          <dc:Bounds x="186" y="142" width="22" height="14" />
        </bpmndi:BPMNLabel>
      </bpmndi:BPMNShape>
      <bpmndi:BPMNShape id="Activity_0111s91_di" bpmnElement="userTask">
        <dc:Bounds x="270" y="77" width="100" height="80" />
        <bpmndi:BPMNLabel />
      </bpmndi:BPMNShape>
      <bpmndi:BPMNShape id="Event_1njqo00_di" bpmnElement="end">
        <dc:Bounds x="432" y="99" width="36" height="36" />
        <bpmndi:BPMNLabel>
          <dc:Bounds x="439" y="142" width="22" height="14" />
        </bpmndi:BPMNLabel>
      </bpmndi:BPMNShape>
      <bpmndi:BPMNEdge id="Flow_13jlccj_di" bpmnElement="Flow_13jlccj">
        <di:waypoint x="215" y="117" />
        <di:waypoint x="270" y="117" />
      </bpmndi:BPMNEdge>
      <bpmndi:BPMNEdge id="Flow_0k8cen8_di" bpmnElement="Flow_0k8cen8">
        <di:waypoint x="370" y="117" />
        <di:waypoint x="432" y="117" />
      </bpmndi:BPMNEdge>
    </bpmndi:BPMNPlane>
  </bpmndi:BPMNDiagram>
</bpmn:definitions>
```
### (4). 原理分析
通过XML,可以剖析出如下对象:

```
# 与流程节点相关
Process
StartEvent
UserTask
EndEvent
SequenceFlow

# 与画布描点相关
BpmnDiagram
BpmnPlane
BPMNShape
BPMNLabel
Bounds
Waypoint
```
### (5). 通过BPMN API生成
```
@Test
public void testBpmn() {
	// 1、创建模型
	BpmnModelInstance modelInstance = Bpmn.createEmptyModel();

	//2、创建BPMN定义元素，设置目标名称空间，并将其添加到新创建的模型实例中
	Definitions definitions = modelInstance.newInstance(Definitions.class);
	definitions.setTargetNamespace(BpmnModelConstants.CAMUNDA_NS);
	definitions.getDomElement().registerNamespace("bpmn", BpmnModelConstants.BPMN20_NS);
	definitions.getDomElement().registerNamespace("bpmndi", BpmnModelConstants.BPMNDI_NS);
	definitions.getDomElement().registerNamespace("dc", BpmnModelConstants.DC_NS);
	definitions.getDomElement().registerNamespace("di", BpmnModelConstants.DI_NS);
	modelInstance.setDefinitions(definitions);

	// 3、向模型新增流程.
	Process process = modelInstance.newInstance(Process.class);
	process.setId("pipeline-demo");
	process.setName("pipelie-demo-example");
	process.setExecutable(true);
	definitions.addChildElement(process);


	// 绘图相关对象
	BpmnDiagram bpmnDiagram = modelInstance.newInstance(BpmnDiagram.class);
	BpmnPlane bpmnPlane = modelInstance.newInstance(BpmnPlane.class);
	bpmnPlane.setBpmnElement(process);

	bpmnDiagram.addChildElement(bpmnPlane);
	definitions.addChildElement(bpmnDiagram);

	// 流程定义里添加开始节点
	StartEvent startEvent = modelInstance.newInstance(StartEvent.class);
	startEvent.setId("start");
	startEvent.setName("开始");
	process.addChildElement(startEvent);

	// 流程定义里添加用户节点
	UserTask userTask = modelInstance.newInstance(UserTask.class);
	userTask.setId("userTask");
	userTask.setName("Task1");
	userTask.setCamundaAssignee("admin");
	process.addChildElement(userTask);

	// 流程定义里添加结束节点
	EndEvent endEvent = modelInstance.newInstance(EndEvent.class);
	endEvent.setId("end");
	endEvent.setName("结束");
	process.addChildElement(endEvent);

	SequenceFlow sequenceFlow1 = modelInstance.newInstance(SequenceFlow.class);
	sequenceFlow1.setId("Flow_13jlccj");
	sequenceFlow1.setSource(startEvent);
	sequenceFlow1.setTarget(userTask);
	process.addChildElement(sequenceFlow1);


	SequenceFlow sequenceFlow2 = modelInstance.newInstance(SequenceFlow.class);
	sequenceFlow2.setId("Flow_0k8cen8");
	sequenceFlow2.setSource(userTask);
	sequenceFlow2.setTarget(endEvent);
	process.addChildElement(sequenceFlow2);

	startEvent.getOutgoing().add(sequenceFlow1);
	userTask.getIncoming().add(sequenceFlow1);
	userTask.getOutgoing().add(sequenceFlow2);

	//
	endEvent.getIncoming().add(sequenceFlow2);


	// 画图(start节点配置)
	Bounds _BPMNShape_StartEvent_2Bounds = modelInstance.newInstance(Bounds.class);
	_BPMNShape_StartEvent_2Bounds.setX(179);
	_BPMNShape_StartEvent_2Bounds.setY(99);
	_BPMNShape_StartEvent_2Bounds.setWidth(36);
	_BPMNShape_StartEvent_2Bounds.setHeight(36);

	BpmnLabel _BPMNShape_StartEvent_2BpmnLabel = modelInstance.newInstance(BpmnLabel.class);
	Bounds _BPMNShape_StartEvent_2BpmnLabelBounds = modelInstance.newInstance(Bounds.class);
	_BPMNShape_StartEvent_2BpmnLabelBounds.setX(186);
	_BPMNShape_StartEvent_2BpmnLabelBounds.setY(142);
	_BPMNShape_StartEvent_2BpmnLabelBounds.setWidth(22);
	_BPMNShape_StartEvent_2BpmnLabelBounds.setHeight(14);
	_BPMNShape_StartEvent_2BpmnLabel.addChildElement(_BPMNShape_StartEvent_2BpmnLabelBounds);

	BpmnShape _BPMNShape_StartEvent_2 = modelInstance.newInstance(BpmnShape.class);
	_BPMNShape_StartEvent_2.setId("_BPMNShape_StartEvent_2");
	_BPMNShape_StartEvent_2.setBpmnElement(startEvent);
	_BPMNShape_StartEvent_2.addChildElement(_BPMNShape_StartEvent_2Bounds);
	_BPMNShape_StartEvent_2.addChildElement(_BPMNShape_StartEvent_2BpmnLabel);
	// 添加绘图
	bpmnPlane.addChildElement(_BPMNShape_StartEvent_2);


	// 画图(userTask)
	Bounds activity_0111s91_diBounds = modelInstance.newInstance(Bounds.class);
	activity_0111s91_diBounds.setX(270);
	activity_0111s91_diBounds.setY(77);
	activity_0111s91_diBounds.setWidth(100);
	activity_0111s91_diBounds.setHeight(80);

	BpmnShape activity_0111s91_di = modelInstance.newInstance(BpmnShape.class);
	activity_0111s91_di.setId("Activity_0111s91_di");
	activity_0111s91_di.setBpmnElement(userTask);
	activity_0111s91_di.addChildElement(activity_0111s91_diBounds);
	// 添加绘图
	bpmnPlane.addChildElement(activity_0111s91_di);


	// 画图(end节点配置)
	Bounds Event_1njqo00_di2Bounds = modelInstance.newInstance(Bounds.class);
	Event_1njqo00_di2Bounds.setX(432);
	Event_1njqo00_di2Bounds.setY(99);
	Event_1njqo00_di2Bounds.setWidth(36);
	Event_1njqo00_di2Bounds.setHeight(36);

	BpmnLabel Event_1njqo00_diBpmnLabel = modelInstance.newInstance(BpmnLabel.class);
	Bounds Event_1njqo00_diBounds = modelInstance.newInstance(Bounds.class);
	Event_1njqo00_diBounds.setX(439);
	Event_1njqo00_diBounds.setY(142);
	Event_1njqo00_diBounds.setWidth(22);
	Event_1njqo00_diBounds.setHeight(14);
	Event_1njqo00_diBpmnLabel.addChildElement(Event_1njqo00_diBounds);

	BpmnShape event_1njqo00_di = modelInstance.newInstance(BpmnShape.class);
	event_1njqo00_di.setId("Event_1njqo00_di");
	event_1njqo00_di.setBpmnElement(endEvent);
	event_1njqo00_di.addChildElement(Event_1njqo00_di2Bounds);
	event_1njqo00_di.addChildElement(Event_1njqo00_diBpmnLabel);
	// 添加绘图
	bpmnPlane.addChildElement(event_1njqo00_di);


	// Flow_13jlccj
	Waypoint flow_13jlccj_di_1_Waypoint = modelInstance.newInstance(Waypoint.class);
	flow_13jlccj_di_1_Waypoint.setX(215);
	flow_13jlccj_di_1_Waypoint.setY(117);

	Waypoint flow_13jlccj_di_2_Waypoint = modelInstance.newInstance(Waypoint.class);
	flow_13jlccj_di_2_Waypoint.setX(270);
	flow_13jlccj_di_2_Waypoint.setY(117);


	BpmnEdge flow_13jlccj_di = modelInstance.newInstance(BpmnEdge.class);
	flow_13jlccj_di.setId("Flow_13jlccj_di");
	flow_13jlccj_di.setBpmnElement(sequenceFlow1);
	flow_13jlccj_di.addChildElement(flow_13jlccj_di_1_Waypoint);
	flow_13jlccj_di.addChildElement(flow_13jlccj_di_2_Waypoint);

	bpmnPlane.addChildElement(flow_13jlccj_di);


	// Flow_0k8cen8_di
	Waypoint flow_0k8cen8_di_1_Waypoint = modelInstance.newInstance(Waypoint.class);
	flow_0k8cen8_di_1_Waypoint.setX(370);
	flow_0k8cen8_di_1_Waypoint.setY(117);

	Waypoint flow_0k8cen8_di_2_Waypoint = modelInstance.newInstance(Waypoint.class);
	flow_0k8cen8_di_2_Waypoint.setX(432);
	flow_0k8cen8_di_2_Waypoint.setY(117);


	BpmnEdge flow_0k8cen8_di = modelInstance.newInstance(BpmnEdge.class);
	flow_0k8cen8_di.setId("Flow_0k8cen8_di");
	flow_0k8cen8_di.setBpmnElement(sequenceFlow2);
	flow_0k8cen8_di.addChildElement(flow_0k8cen8_di_1_Waypoint);
	flow_0k8cen8_di.addChildElement(flow_0k8cen8_di_2_Waypoint);
	bpmnPlane.addChildElement(flow_0k8cen8_di);

	System.out.println(Bpmn.convertToString(modelInstance));
}
```
### (6). 最后生成的xml文件如下
```
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<definitions xmlns:bpmn="http://www.omg.org/spec/BPMN/20100524/MODEL" xmlns:bpmndi="http://www.omg.org/spec/BPMN/20100524/DI" xmlns:camunda="http://camunda.org/schema/1.0/bpmn" xmlns:dc="http://www.omg.org/spec/DD/20100524/DC" xmlns:di="http://www.omg.org/spec/DD/20100524/DI" id="definitions_b292ff24-fe48-4348-8635-54cf5d82ced6" targetNamespace="http://camunda.org/schema/1.0/bpmn" xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL">
  <process id="pipeline-demo" isExecutable="true" name="pipelie-demo-example">
    <startEvent id="start" name="开始">
      <outgoing>Flow_13jlccj</outgoing>
    </startEvent>
    <userTask camunda:assignee="admin" id="userTask" name="Task1">
      <incoming>Flow_13jlccj</incoming>
      <outgoing>Flow_0k8cen8</outgoing>
    </userTask>
    <endEvent id="end" name="结束">
      <incoming>Flow_0k8cen8</incoming>
    </endEvent>
    <sequenceFlow id="Flow_13jlccj" sourceRef="start" targetRef="userTask"/>
    <sequenceFlow id="Flow_0k8cen8" sourceRef="userTask" targetRef="end"/>
  </process>
  <bpmndi:BPMNDiagram id="BPMNDiagram_2520f04f-4a39-4c22-ace0-2314cfb1170e">
    <bpmndi:BPMNPlane bpmnElement="pipeline-demo" id="BPMNPlane_94dce98b-afac-48fb-9e57-b3692decb051">
      <bpmndi:BPMNShape bpmnElement="start" id="_BPMNShape_StartEvent_2">
        <dc:Bounds height="36.0" width="36.0" x="179.0" y="99.0"/>
        <bpmndi:BPMNLabel id="BPMNLabel_17f8f5c1-b23a-4302-b63a-740f7ebcd965">
          <dc:Bounds height="14.0" width="22.0" x="186.0" y="142.0"/>
        </bpmndi:BPMNLabel>
      </bpmndi:BPMNShape>
      <bpmndi:BPMNShape bpmnElement="userTask" id="Activity_0111s91_di">
        <dc:Bounds height="80.0" width="100.0" x="270.0" y="77.0"/>
      </bpmndi:BPMNShape>
      <bpmndi:BPMNShape bpmnElement="end" id="Event_1njqo00_di">
        <dc:Bounds height="36.0" width="36.0" x="432.0" y="99.0"/>
        <bpmndi:BPMNLabel id="BPMNLabel_3e5a09a0-5a48-4589-b218-d7231cb9aa2c">
          <dc:Bounds height="14.0" width="22.0" x="439.0" y="142.0"/>
        </bpmndi:BPMNLabel>
      </bpmndi:BPMNShape>
      <bpmndi:BPMNEdge bpmnElement="Flow_13jlccj" id="Flow_13jlccj_di">
        <di:waypoint x="215.0" y="117.0"/>
        <di:waypoint x="270.0" y="117.0"/>
      </bpmndi:BPMNEdge>
      <bpmndi:BPMNEdge bpmnElement="Flow_0k8cen8" id="Flow_0k8cen8_di">
        <di:waypoint x="370.0" y="117.0"/>
        <di:waypoint x="432.0" y="117.0"/>
      </bpmndi:BPMNEdge>
    </bpmndi:BPMNPlane>
  </bpmndi:BPMNDiagram>
</definitions>
```
### (7). Camunda流式API生成案例
```
@Test
public void testBpmnToXml() {
	// @formatter:off
	BpmnModelInstance modelInstance = Bpmn.createProcess()
			.startEvent()
				.userTask()
				.id("task1")
				.name("task1")
				.parallelGateway("fork")
					.serviceTask()
				.parallelGateway("join")
				.moveToNode("fork")
					.userTask()
				.connectTo("join")
				.moveToNode("fork")
					.scriptTask()
				.connectTo("join")
			.endEvent()
			.done();
	// @formatter:on

	// 找到某个节点,配置naemspace和属性
	ModelElementInstance task = modelInstance.getModelElementById("task1");
	task.setAttributeValueNs("http://lixin.help", "hello", "world");
	String hello = task.getAttributeValueNs("http://lixin.help", "hello");

	if (task instanceof FlowNode) {
		FlowNode flowNode = (FlowNode) task;
		ExtensionElements extensionElements = flowNode.getExtensionElements();
		if (null == extensionElements) {
			extensionElements = modelInstance.newInstance(ExtensionElements.class);
			ModelElementInstance myExtensionElement = extensionElements.addExtensionElement("http://lixin.help", "myExtensionElement");
			myExtensionElement.setAttributeValue("testName", "testValue");
			flowNode.setExtensionElements(extensionElements);
		}

		List<ModelElementInstance> list = extensionElements.getElementsQuery().filterByType(ModelElementInstance.class).list();
		for (ModelElementInstance modelElementInstance : list) {
			// @formatter:off
			if (null != modelElementInstance.getElementType() &&
					null != modelElementInstance.getElementType().getTypeName() &&
					modelElementInstance.getElementType().getTypeName().equals("myExtensionElement")) {
				// @formatter:on
				String testName = modelElementInstance.getAttributeValueNs("http://lixin.help", "testName");
				System.out.println(testName);
			}
		}
		// Collection<SequenceFlow> incoming = flowNode.getIncoming();
		// Collection<SequenceFlow> outgoing = flowNode.getOutgoing();
	}
	System.out.println(Bpmn.convertToString(modelInstance));
}
```
### (8). 总结
Camunda的设计还是挺好的,基本上xml节点名称,和我们的类名称是一一对应的,不费吹灰之力,就可以实现手写xml生成过程.  