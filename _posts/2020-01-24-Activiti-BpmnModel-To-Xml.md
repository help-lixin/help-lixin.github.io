---
layout: post
title: 'Activiti源码(手动生成xml)'
date: 2020-01-24
author: 李新
tags: Activiti
---

### (1). 需求
有这样一个需求,需要通过代码的方式,创建bpmn.xml文件.

### (2). java代码
```
package org.activiti.examples;

import org.activiti.bpmn.BpmnAutoLayout;
import org.activiti.bpmn.converter.BpmnXMLConverter;
import org.activiti.bpmn.model.*;
import org.activiti.validation.ValidationError;
import org.activiti.validation.validator.impl.BpmnModelValidator;
import org.apache.commons.lang3.ObjectUtils;

import java.util.ArrayList;
import java.util.List;

public class Test {

    @org.junit.jupiter.api.Test
    public void test() {

        BpmnModel bpmnModel = new BpmnModel();

        //设置流程信息
        //此信息都可以通过前期自定义数据,使用时再查询
        org.activiti.bpmn.model.Process process = new org.activiti.bpmn.model.Process();
        process.setId("test_model_3");
        process.setName("测试流程图三");
        process.setExecutable(true);

        bpmnModel.addProcess(process);


        //添加流程节点信息---start
        String startId = "startEvent_id_1";
        String startName = "开始_1";

        FlowNode startFlowElement = createStartFlowElement(startId, startName);

        // 用户节点
        String userTaskId = "userTask_id_1";
        String userTaskName = "领导审批";
        FlowNode admin = createCommonUserTask(userTaskId, userTaskName, "admin");


        // 结束节点
        String endId = "endEvent_id_1";
        String endName = "结束_1";

        FlowNode endFlowElement = createEndFlowElement(endId, endName);

        // 连线1
        String name = "";
        String startToUserTaskId = "startEvent_id_1_to_userTask_id_1";
        String startToUserTaskIdSource = startId;
        String startToUserTaskIdTarget = userTaskId;
        SequenceFlow sequeneFlow1 = createSequeneFlow(startToUserTaskId, name, startToUserTaskIdSource, startToUserTaskIdTarget);


        // 连线2
        String userTaskIdToEnd = "userTask_id_1_to_endEvent_id_1";
        String userTaskIdToEndSource = userTaskId;
        String userTaskIdToEndTarget = endId;
        SequenceFlow sequeneFlow2 = createSequeneFlow(userTaskIdToEnd, name, userTaskIdToEndSource, userTaskIdToEndTarget);


        startFlowElement.getOutgoingFlows().add(sequeneFlow1);

        admin.getIncomingFlows().add(sequeneFlow1);
        admin.getOutgoingFlows().add(sequeneFlow2);

        endFlowElement.getIncomingFlows().add(sequeneFlow2);

        process.addFlowElement(startFlowElement);
        process.addFlowElement(admin);
        process.addFlowElement(endFlowElement);
        process.addFlowElement(sequeneFlow1);
        process.addFlowElement(sequeneFlow2);

        List<ValidationError> errors = new ArrayList<>();

        BpmnModelValidator bpmnModelValidator = new BpmnModelValidator();
        bpmnModelValidator.validate(bpmnModel, errors);
        if (errors.size() > 0) {
            System.out.println();
        }
        //生成自动布局
        new BpmnAutoLayout(bpmnModel).execute();
        BpmnXMLConverter bpmnXMLConverter = new BpmnXMLConverter();
        byte[] xml = bpmnXMLConverter.convertToXML(bpmnModel, "UTF-8");
        System.out.println(new String(xml));
    }

    public SequenceFlow createSequeneFlow(String id, String name, String sourceId, String targetId) {
        SequenceFlow sequenceFlow = new SequenceFlow();
        sequenceFlow.setId(id);
        sequenceFlow.setName(name);
        if (ObjectUtils.isNotEmpty(targetId)) {
            sequenceFlow.setTargetRef(targetId);
        }
        if (ObjectUtils.isNotEmpty(sourceId)) {
            sequenceFlow.setSourceRef(sourceId);
        }
        return sequenceFlow;
    }

    public FlowNode createCommonUserTask(String id, String name, String assignee) {
        UserTask userTask = new UserTask();
        userTask.setId(id);
        userTask.setName(name);
        userTask.setAssignee(assignee);
        userTask.getIncomingFlows();
        return userTask;
    }

    public FlowNode createStartFlowElement(String id, String name) {
        StartEvent startEvent = new StartEvent();
        startEvent.setId(id);
        startEvent.setName(name);
        return startEvent;
    }

    public FlowNode createEndFlowElement(String id, String name) {
        EndEvent endEvent = new EndEvent();
        endEvent.setId(id);
        endEvent.setName(name);
        return endEvent;
    }

}
```
### (3). 生成xml内容
```
<?xml version="1.0" encoding="UTF-8"?>
<bpmn2:definitions xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL" xmlns:bpmn2="http://www.omg.org/spec/BPMN/20100524/MODEL" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:activiti="http://activiti.org/bpmn" xmlns:bpmndi="http://www.omg.org/spec/BPMN/20100524/DI" xmlns:omgdc="http://www.omg.org/spec/DD/20100524/DC" xmlns:omgdi="http://www.omg.org/spec/DD/20100524/DI" typeLanguage="http://www.w3.org/2001/XMLSchema" expressionLanguage="http://www.w3.org/1999/XPath" targetNamespace="http://www.activiti.org/test">
  <bpmn2:process id="test_model_3" name="测试流程图三" isExecutable="true">
    <bpmn2:startEvent id="startEvent_id_1" name="开始_1">
      <bpmn2:outgoing>startEvent_id_1_to_userTask_id_1</bpmn2:outgoing>
    </bpmn2:startEvent>
    <bpmn2:userTask id="userTask_id_1" name="领导审批" activiti:assignee="admin">
      <bpmn2:incoming>startEvent_id_1_to_userTask_id_1</bpmn2:incoming>
      <bpmn2:outgoing>userTask_id_1_to_endEvent_id_1</bpmn2:outgoing>
    </bpmn2:userTask>
    <bpmn2:endEvent id="endEvent_id_1" name="结束_1">
      <bpmn2:incoming>userTask_id_1_to_endEvent_id_1</bpmn2:incoming>
    </bpmn2:endEvent>
    <bpmn2:sequenceFlow id="startEvent_id_1_to_userTask_id_1" sourceRef="startEvent_id_1" targetRef="userTask_id_1"></bpmn2:sequenceFlow>
    <bpmn2:sequenceFlow id="userTask_id_1_to_endEvent_id_1" sourceRef="userTask_id_1" targetRef="endEvent_id_1"></bpmn2:sequenceFlow>
  </bpmn2:process>
  <bpmndi:BPMNDiagram id="BPMNDiagram_test_model_3">
    <bpmndi:BPMNPlane bpmnElement="test_model_3" id="BPMNPlane_test_model_3">
      <bpmndi:BPMNShape bpmnElement="endEvent_id_1" id="BPMNShape_endEvent_id_1">
        <omgdc:Bounds height="30.0" width="30.0" x="230.0" y="15.0"></omgdc:Bounds>
      </bpmndi:BPMNShape>
      <bpmndi:BPMNShape bpmnElement="startEvent_id_1" id="BPMNShape_startEvent_id_1">
        <omgdc:Bounds height="30.0" width="30.0" x="0.0" y="15.0"></omgdc:Bounds>
      </bpmndi:BPMNShape>
      <bpmndi:BPMNShape bpmnElement="userTask_id_1" id="BPMNShape_userTask_id_1">
        <omgdc:Bounds height="60.0" width="100.0" x="80.0" y="0.0"></omgdc:Bounds>
      </bpmndi:BPMNShape>
      <bpmndi:BPMNEdge bpmnElement="userTask_id_1_to_endEvent_id_1" id="BPMNEdge_userTask_id_1_to_endEvent_id_1">
        <omgdi:waypoint x="180.0" y="30.0"></omgdi:waypoint>
        <omgdi:waypoint x="230.0" y="30.0"></omgdi:waypoint>
      </bpmndi:BPMNEdge>
      <bpmndi:BPMNEdge bpmnElement="startEvent_id_1_to_userTask_id_1" id="BPMNEdge_startEvent_id_1_to_userTask_id_1">
        <omgdi:waypoint x="30.0" y="30.0"></omgdi:waypoint>
        <omgdi:waypoint x="80.0" y="30.0"></omgdi:waypoint>
      </bpmndi:BPMNEdge>
    </bpmndi:BPMNPlane>
  </bpmndi:BPMNDiagram>
</bpmn2:definitions>
```
