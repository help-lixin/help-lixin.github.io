---
layout: post
title: 'Zeebe Client简单入门(三)' 
date: 2022-02-02
author: 李新
tags:  Zeebe
---

### (1). 概述
在前面,运行起来了Broker,在这里,需要有一个简单的Demo来运行,demo来自于zeebe的github.

### (2). Zeebe Sample
!["Zeebe Sample"](/assets/zeebe/imgs/zeebe-samples.png)

### (3). 流程定义(demoProcessSingleTask.bpmn)
```
<?xml version="1.0" encoding="UTF-8"?>
<bpmn:definitions xmlns:bpmn="http://www.omg.org/spec/BPMN/20100524/MODEL" xmlns:bpmndi="http://www.omg.org/spec/BPMN/20100524/DI" xmlns:di="http://www.omg.org/spec/DD/20100524/DI" xmlns:dc="http://www.omg.org/spec/DD/20100524/DC" xmlns:zeebe="http://camunda.org/schema/zeebe/1.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:camunda="http://camunda.org/schema/1.0/bpmn" id="Definitions_1" targetNamespace="http://bpmn.io/schema/bpmn" exporter="Zeebe Modeler" exporterVersion="0.8.0">
  <bpmn:process id="demoProcessSingleTask" isExecutable="true">
    <bpmn:startEvent id="start" name="start">
      <bpmn:outgoing>SequenceFlow_1sz6737</bpmn:outgoing>
    </bpmn:startEvent>
    <bpmn:sequenceFlow id="SequenceFlow_1sz6737" sourceRef="start" targetRef="taskA" />
    <bpmn:sequenceFlow id="SequenceFlow_06ytcxw" sourceRef="taskA" targetRef="end" />
    <bpmn:endEvent id="end" name="end">
      <bpmn:incoming>SequenceFlow_06ytcxw</bpmn:incoming>
    </bpmn:endEvent>
    <bpmn:serviceTask id="taskA" name="task A">
      <bpmn:extensionElements>
        <zeebe:taskDefinition type="foo" />
      </bpmn:extensionElements>
      <bpmn:incoming>SequenceFlow_1sz6737</bpmn:incoming>
      <bpmn:outgoing>SequenceFlow_06ytcxw</bpmn:outgoing>
    </bpmn:serviceTask>
  </bpmn:process>
  <bpmndi:BPMNDiagram id="BPMNDiagram_1">
    <bpmndi:BPMNPlane id="BPMNPlane_1" bpmnElement="demoProcess">
      <bpmndi:BPMNShape id="_BPMNShape_StartEvent_2" bpmnElement="start">
        <dc:Bounds x="173" y="102" width="36" height="36" />
        <bpmndi:BPMNLabel>
          <dc:Bounds x="180" y="138" width="22" height="12" />
        </bpmndi:BPMNLabel>
      </bpmndi:BPMNShape>
      <bpmndi:BPMNEdge id="SequenceFlow_1sz6737_di" bpmnElement="SequenceFlow_1sz6737">
        <di:waypoint x="209" y="120" />
        <di:waypoint x="310" y="120" />
        <bpmndi:BPMNLabel>
          <dc:Bounds x="260" y="105" width="0" height="0" />
        </bpmndi:BPMNLabel>
      </bpmndi:BPMNEdge>
      <bpmndi:BPMNEdge id="SequenceFlow_06ytcxw_di" bpmnElement="SequenceFlow_06ytcxw">
        <di:waypoint x="410" y="120" />
        <di:waypoint x="542" y="120" />
        <bpmndi:BPMNLabel>
          <dc:Bounds x="456" y="105" width="0" height="0" />
        </bpmndi:BPMNLabel>
      </bpmndi:BPMNEdge>
      <bpmndi:BPMNShape id="EndEvent_0gbv3sc_di" bpmnElement="end">
        <dc:Bounds x="542" y="102" width="36" height="36" />
        <bpmndi:BPMNLabel>
          <dc:Bounds x="551" y="138" width="19" height="14" />
        </bpmndi:BPMNLabel>
      </bpmndi:BPMNShape>
      <bpmndi:BPMNShape id="ServiceTask_09m0goq_di" bpmnElement="taskA">
        <dc:Bounds x="310" y="80" width="100" height="80" />
      </bpmndi:BPMNShape>
    </bpmndi:BPMNPlane>
  </bpmndi:BPMNDiagram>
</bpmn:definitions>
```
### (4). 流程部署(ProcessDeployer)
```
package io.camunda.zeebe.example.process;

import io.camunda.zeebe.client.ZeebeClient;
import io.camunda.zeebe.client.ZeebeClientBuilder;
import io.camunda.zeebe.client.api.response.DeploymentEvent;


public final class ProcessDeployer {

  public static void main(final String[] args) {
    final String defaultAddress = "localhost:26500";
    final String envVarAddress = System.getenv("ZEEBE_ADDRESS");

    final ZeebeClientBuilder clientBuilder;
    if (envVarAddress != null) {
      /* Connect to Camunda Cloud Cluster, assumes that credentials are set in environment variables.
       * See JavaDoc on class level for details
       */
      clientBuilder = ZeebeClient.newClientBuilder().gatewayAddress(envVarAddress);
    } else {
      // connect to local deployment; assumes that authentication is disabled
      clientBuilder = ZeebeClient.newClientBuilder().gatewayAddress(defaultAddress).usePlaintext();
    }

    try (final ZeebeClient client = clientBuilder.build()) {

	  // *****************************************************************************	
	  // 部署流程实例
	  // *****************************************************************************	
      final DeploymentEvent deploymentEvent = client.newDeployCommand().addResourceFromClasspath("demoProcessSingleTask.bpmn").send().join();

      System.out.println("Deployment created with key: " + deploymentEvent.getKey());
    }
  }
}
```
### (5). 流程运行(ProcessInstanceWithResultCreator)
```
package io.camunda.zeebe.example.process;

import io.camunda.zeebe.client.ZeebeClient;
import io.camunda.zeebe.client.ZeebeClientBuilder;
import io.camunda.zeebe.client.api.response.ProcessInstanceResult;
import java.time.Duration;
import java.util.Map;

public class ProcessInstanceWithResultCreator {
  public static void main(final String[] args) {
    final String defaultAddress = "localhost:26500";
    final String envVarAddress = System.getenv("ZEEBE_ADDRESS");

    final ZeebeClientBuilder clientBuilder;
    if (envVarAddress != null) {
      clientBuilder = ZeebeClient.newClientBuilder().gatewayAddress(envVarAddress);
    } else {
      clientBuilder = ZeebeClient.newClientBuilder().gatewayAddress(defaultAddress).usePlaintext();
    }

    final String bpmnProcessId = "demoProcessSingleTask";

    try (final ZeebeClient client = clientBuilder.build()) {

      // open job workers so that task are executed and process is completed
      openJobWorker(client); 
      System.out.println("Creating process instance");

      final ProcessInstanceResult processInstanceResult =
          client
              .newCreateInstanceCommand()
              .bpmnProcessId(bpmnProcessId)
              .latestVersion()
              .withResult() // to await the completion of process execution and return result
              .send()
              .join();

      System.out.println(
          "Process instance created with key: "
              + processInstanceResult.getProcessInstanceKey()
              + " and completed with results: "
              + processInstanceResult.getVariables());
    }
  }

  private static void openJobWorker(final ZeebeClient client) {
    client
        .newWorker()
        .jobType("foo")
        .handler(
            (jobClient, job) ->
                jobClient
                    .newCompleteCommand(job.getKey())
                    .variables(Map.of("job", job.getKey()))
                    .send())
        .timeout(Duration.ofSeconds(10))
        .open();
  }
}
```
### (6). Kibana查看索引信息
!["Kibana索引信息"](/assets/zeebe/imgs/kibana-zeebe.png)
