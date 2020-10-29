---
layout: post
title: 'Activiti源码(RepositoryService)'
date: 2020-01-24
author: 李新
tags: Activiti
---

### (1).流程部署
> Activiti中流程部署入口如下:RepositoryService.deploy

### (2).RepositoryService.createDeployment
```
public interface RepositoryService {
    // 创建流程部署
    DeploymentBuilder createDeployment();
}
```
### (3).DeploymentBuilder介绍
```
public interface DeploymentBuilder {
  // 资源名称(resourceName)对应:ACT_GE_BYTEARRAY表NAME_
  // inputStream对应ACT_GE_BYTEARRAY表BYTES_
  DeploymentBuilder addInputStream(String resourceName, InputStream inputStream);

  DeploymentBuilder addClasspathResource(String resource);

  DeploymentBuilder addString(String resourceName, String text);
  
  DeploymentBuilder addBytes(String resourceName, byte[] bytes);

  DeploymentBuilder addZipInputStream(ZipInputStream zipInputStream);

  DeploymentBuilder addBpmnModel(String resourceName, BpmnModel bpmnModel);
  
  Deployment deploy();
}
```
### (4).流程定义效果图

!["流程定义"](/assets/activiti/imgs/flow-define.png)

### (5).流程定义XML
```
<?xml version="1.0" encoding="UTF-8"?>
<definitions xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:activiti="http://activiti.org/bpmn" xmlns:bpmndi="http://www.omg.org/spec/BPMN/20100524/DI" xmlns:omgdc="http://www.omg.org/spec/DD/20100524/DC" xmlns:omgdi="http://www.omg.org/spec/DD/20100524/DI" typeLanguage="http://www.w3.org/2001/XMLSchema" expressionLanguage="http://www.w3.org/1999/XPath" targetNamespace="http://www.activiti.org/test">
  <process id="process1" name="My process" isExecutable="true">
    <startEvent id="startevent1" name="Start"></startEvent>
    <userTask id="usertask1" name="任务节点1" activiti:assignee="lixin"></userTask>
    <sequenceFlow id="flow1" sourceRef="startevent1" targetRef="usertask1"></sequenceFlow>
    <userTask id="usertask2" name="任务节点2"></userTask>
    <sequenceFlow id="flow3" sourceRef="usertask1" targetRef="usertask2"></sequenceFlow>
    <endEvent id="endevent1" name="End"></endEvent>
    <sequenceFlow id="flow4" sourceRef="usertask2" targetRef="endevent1"></sequenceFlow>
  </process>
  <bpmndi:BPMNDiagram id="BPMNDiagram_process1">
    <bpmndi:BPMNPlane bpmnElement="process1" id="BPMNPlane_process1">
      <bpmndi:BPMNShape bpmnElement="startevent1" id="BPMNShape_startevent1">
        <omgdc:Bounds height="35.0" width="35.0" x="130.0" y="170.0"></omgdc:Bounds>
      </bpmndi:BPMNShape>
      <bpmndi:BPMNShape bpmnElement="usertask1" id="BPMNShape_usertask1">
        <omgdc:Bounds height="55.0" width="105.0" x="210.0" y="160.0"></omgdc:Bounds>
      </bpmndi:BPMNShape>
      <bpmndi:BPMNShape bpmnElement="usertask2" id="BPMNShape_usertask2">
        <omgdc:Bounds height="55.0" width="105.0" x="360.0" y="158.0"></omgdc:Bounds>
      </bpmndi:BPMNShape>
      <bpmndi:BPMNShape bpmnElement="endevent1" id="BPMNShape_endevent1">
        <omgdc:Bounds height="35.0" width="35.0" x="510.0" y="168.0"></omgdc:Bounds>
      </bpmndi:BPMNShape>
      <bpmndi:BPMNEdge bpmnElement="flow1" id="BPMNEdge_flow1">
        <omgdi:waypoint x="165.0" y="187.0"></omgdi:waypoint>
        <omgdi:waypoint x="210.0" y="187.0"></omgdi:waypoint>
      </bpmndi:BPMNEdge>
      <bpmndi:BPMNEdge bpmnElement="flow3" id="BPMNEdge_flow3">
        <omgdi:waypoint x="315.0" y="187.0"></omgdi:waypoint>
        <omgdi:waypoint x="360.0" y="185.0"></omgdi:waypoint>
      </bpmndi:BPMNEdge>
      <bpmndi:BPMNEdge bpmnElement="flow4" id="BPMNEdge_flow4">
        <omgdi:waypoint x="465.0" y="185.0"></omgdi:waypoint>
        <omgdi:waypoint x="510.0" y="185.0"></omgdi:waypoint>
      </bpmndi:BPMNEdge>
    </bpmndi:BPMNPlane>
  </bpmndi:BPMNDiagram>
</definitions>
```
### (6).activiti.cfg.xml
```
<?xml version="1.0" encoding="UTF-8"?>

<beans xmlns="http://www.springframework.org/schema/beans"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.springframework.org/schema/beans   http://www.springframework.org/schema/beans/spring-beans.xsd">

  <bean id="processEngineConfiguration" class="org.activiti.engine.impl.cfg.StandaloneProcessEngineConfiguration">
  	<property name="dataSource" ref="dataSource"/>
    <property name="databaseSchemaUpdate" value="drop-create" />
  </bean>
  
  <bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource" init-method="init" destroy-method="close">
  		<!-- 基本属性 url、user、password -->
        <property name="url" value="jdbc:mysql://localhost:3306/activiti" />
        <property name="username" value="root" />
        <property name="password" value="123456" />
        <!-- 配置初始化大小、最小、最大 -->
        <property name="initialSize" value="5" />
        <property name="minIdle" value="5" />
        <property name="maxActive" value="10" />
        <!-- 配置从连接池获取连接等待超时的时间 -->
        <property name="maxWait" value="10000" />
        <property name="timeBetweenEvictionRunsMillis" value="600000" />
        <!-- 配置一个连接在池中最大空闲时间，单位是毫秒 -->
        <property name="minEvictableIdleTimeMillis" value="300000" />
        <!-- 设置从连接池获取连接时是否检查连接有效性，true时，每次都检查;false时，不检查 -->
        <property name="testOnBorrow" value="false" />
        <!-- 设置往连接池归还连接时是否检查连接有效性，true时，每次都检查;false时，不检查 -->
        <property name="testOnReturn" value="false" />
        <!-- 设置从连接池获取连接时是否检查连接有效性，true时，如果连接空闲时间超过minEvictableIdleTimeMillis进行检查，否则不检查;false时，不检查 -->
        <property name="testWhileIdle" value="true" />
        <!-- 检验连接是否有效的查询语句。如果数据库Driver支持ping()方法，则优先使用ping()方法进行检查，否则使用validationQuery查询进行检查。(Oracle jdbc Driver目前不支持ping方法) -->
        <property name="validationQuery" value="SELECT 1" />
        <!-- 打开PSCache，并且指定每个连接上PSCache的大小，Oracle等支持游标的数据库，打开此开关，会以数量级提升性能，具体查阅PSCache相关资料 -->
        <property name="poolPreparedStatements" value="true" />
        <property name="maxPoolPreparedStatementPerConnectionSize" value="20" />
        <property name="filters" value="stat,slf4j" />   
  </bean>
</beans>
```
### (7).流程部署
```
package help.lixin.activiti;

import static org.junit.Assert.assertNotNull;

import java.net.URL;
import java.util.Arrays;

import org.activiti.bpmn.model.BpmnModel;
import org.activiti.bpmn.model.EndEvent;
import org.activiti.bpmn.model.Process;
import org.activiti.bpmn.model.SequenceFlow;
import org.activiti.bpmn.model.StartEvent;
import org.activiti.bpmn.model.UserTask;
import org.activiti.engine.ProcessEngine;
import org.activiti.engine.ProcessEngines;
import org.activiti.engine.RepositoryService;
import org.activiti.engine.repository.Deployment;
import org.junit.Test;

public class UserTaskTest {

	@Test
	public void testDeployInputStream() throws Exception {
		ProcessEngine processEngine = ProcessEngines.getDefaultProcessEngine();
		URL url = UserTaskTest.class.getClassLoader().getResource("user-task.bpmn");
		RepositoryService repositoryService = processEngine.getRepositoryService();
		// 流程部署时,会往两个表中存储数据(ACT_RE_DEPLOYMENT/ACT_GE_BYTEARRAY)
		Deployment deployment = repositoryService.createDeployment() //
				.name("用户任务") // ACT_RE_DEPLOYMENT.NAME_
				.category("OA") // ACT_RE_DEPLOYMENT.CATEGORY_
				.tenantId("000007") // ACT_RE_DEPLOYMENT.TENANTID_
				.addInputStream("user-task", url.openStream()) 
				// ACT_GE_BYTEARRAY.NAME_ /  ACT_GE_BYTEARRAY.BYTES_
				.deploy();
		assertNotNull("deployment object is not null", deployment);
	} //end testDeployInputStream

	@Test
	public void testDeployBpmnModel() {
		// 创建节点
		SequenceFlow flow1 = new SequenceFlow();
		flow1.setId("flow1");
		flow1.setName("开始节点 -> 任务节点1");
		flow1.setSourceRef("start");
		flow1.setTargetRef("userTask1");

		// 创建节点
		SequenceFlow flow2 = new SequenceFlow();
		flow2.setId("flow2");
		flow2.setName("任务节点1 -> 任务节点2");
		flow2.setSourceRef("userTask1");
		flow2.setTargetRef("userTask2");
		
		// 创建节点
		SequenceFlow flow3 = new SequenceFlow();
		flow3.setId("flow3");
		flow3.setName("任务节点2 -> 结束节点");
		flow3.setSourceRef("userTask2");
		flow3.setTargetRef("end");
		
		// 创建开始节点
		StartEvent startEvent = new StartEvent();
		startEvent.setId("start");
		startEvent.setName("开始节点");
		startEvent.setOutgoingFlows(Arrays.asList(flow1));

		UserTask userTask1 = new UserTask();
		userTask1.setId("userTask1");
		userTask1.setName("任务节点1");
		userTask1.setIncomingFlows(Arrays.asList(flow1));
		userTask1.setOutgoingFlows(Arrays.asList(flow2));
		
		
		UserTask userTask2 = new UserTask();
		userTask2.setId("userTask2");
		userTask2.setName("任务节点2");
		userTask2.setIncomingFlows(Arrays.asList(flow2));
		userTask2.setOutgoingFlows(Arrays.asList(flow3));
		
		EndEvent endEvent = new EndEvent();
		endEvent.setId("end");
		endEvent.setName("结束节点");
		endEvent.setIncomingFlows(Arrays.asList(flow3));
		

		String resource = "user-task-addBpmnModel";
		BpmnModel model = new BpmnModel();
		// 创建流程
		org.activiti.bpmn.model.Process process = new Process();
		process.setId("process1");
		
		process.addFlowElement(startEvent);
		process.addFlowElement(flow1);
		process.addFlowElement(userTask1);
		process.addFlowElement(flow2);
		process.addFlowElement(userTask2);
		process.addFlowElement(flow3);
		process.addFlowElement(endEvent);
		
		// 添加流程
		model.addProcess(process);
		
		ProcessEngine processEngine = ProcessEngines.getDefaultProcessEngine();
		RepositoryService repositoryService = processEngine.getRepositoryService();
		Deployment deployment = repositoryService.createDeployment()
		                 .name(resource)
					     .addBpmnModel(resource, model)
					     .deploy();
		assertNotNull("deployment object is not null", deployment);
	} //end testDeployBpmnModel
 
}
```
### (8).流程部署表结构
!["ACT_GE_BYTEARRAY"](/assets/activiti/imgs/flow-deploy-table-1.png)

```
CREATE TABLE `ACT_GE_BYTEARRAY` (
  `ID_` varchar(64) COLLATE utf8_bin NOT NULL DEFAULT '' COMMENT '主键',
  `REV_` int(11) DEFAULT NULL COMMENT  '版本',
  `NAME_` varchar(255) COLLATE utf8_bin DEFAULT NULL COMMENT  '资源名称(resourceName)',
  `DEPLOYMENT_ID_` varchar(64) COLLATE utf8_bin DEFAULT NULL COMMENT  '外键ACT_RE_DEPLOYMENT表ID',
  `BYTES_` longblob COMMENT '流程文档内容',
  `GENERATED_` tinyint(4) DEFAULT NULL COMMENT '',
  PRIMARY KEY (`ID_`),
  KEY `ACT_FK_BYTEARR_DEPL` (`DEPLOYMENT_ID_`),
  CONSTRAINT `ACT_FK_BYTEARR_DEPL` FOREIGN KEY (`DEPLOYMENT_ID_`) REFERENCES `ACT_RE_DEPLOYMENT` (`ID_`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT '通用字节数组存储';
```

!["ACT_RE_DEPLOYMENT"](/assets/activiti/imgs/flow-deploy-table-2.png)

```
CREATE TABLE `ACT_RE_DEPLOYMENT` (
  `ID_` varchar(64) COLLATE utf8_bin NOT NULL DEFAULT '' COMMENT '主键',
  `NAME_` varchar(255) COLLATE utf8_bin DEFAULT NULL COMMENT '部署名称',
  `CATEGORY_` varchar(255) COLLATE utf8_bin DEFAULT NULL  COMMENT '类别',
  `KEY_` varchar(255) COLLATE utf8_bin DEFAULT NULL  COMMENT '主键',
  `TENANT_ID_` varchar(255) COLLATE utf8_bin DEFAULT ''  COMMENT '多租户唯一ID',
  `DEPLOY_TIME_` timestamp(3) NULL DEFAULT NULL COMMENT  '部署时间',
  `ENGINE_VERSION_` varchar(255) COLLATE utf8_bin DEFAULT NULL COMMENT '版本',
  PRIMARY KEY (`ID_`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin;

```

!["ACT_RE_PROCDEF"](/assets/activiti/imgs/flow-deploy-table-3.png)

```
CREATE TABLE `ACT_RE_PROCDEF` (
  `ID_` varchar(64) COLLATE utf8_bin NOT NULL,
  `REV_` int(11) DEFAULT NULL,
  `CATEGORY_` varchar(255) COLLATE utf8_bin DEFAULT NULL,
  `NAME_` varchar(255) COLLATE utf8_bin DEFAULT NULL,
  `KEY_` varchar(255) COLLATE utf8_bin NOT NULL,
  `VERSION_` int(11) NOT NULL,
  `DEPLOYMENT_ID_` varchar(64) COLLATE utf8_bin DEFAULT NULL COMMENT '流程部署ID',
  `RESOURCE_NAME_` varchar(4000) COLLATE utf8_bin DEFAULT NULL,
  `DGRM_RESOURCE_NAME_` varchar(4000) COLLATE utf8_bin DEFAULT NULL,
  `DESCRIPTION_` varchar(4000) COLLATE utf8_bin DEFAULT NULL,
  `HAS_START_FORM_KEY_` tinyint(4) DEFAULT NULL,
  `HAS_GRAPHICAL_NOTATION_` tinyint(4) DEFAULT NULL,
  `SUSPENSION_STATE_` int(11) DEFAULT NULL,
  `TENANT_ID_` varchar(255) COLLATE utf8_bin DEFAULT '',
  `ENGINE_VERSION_` varchar(255) COLLATE utf8_bin DEFAULT NULL,
  PRIMARY KEY (`ID_`),
  UNIQUE KEY `ACT_UNIQ_PROCDEF` (`KEY_`,`VERSION_`,`TENANT_ID_`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin;
```

### (9).校验BpmnModel实例对象
```
@Test
public void testValidator() {
    // 创建流程
    BpmnModel model = createBpmnModel();
    // 创建流程验证工厂
    ProcessValidatorFactory factory = new ProcessValidatorFactory();
    // 创建流程验证
    ProcessValidator processValidator = factory.createDefaultProcessValidator();
    // 验证失败结果集
    List<ValidationError> validatorResult = processValidator.validate(model);
    Assert.assertTrue(validatorResult.isEmpty());
} //end testValidator

private BpmnModel createBpmnModel() {
    // 创建节点
    SequenceFlow flow1 = new SequenceFlow();
    flow1.setId("flow1");
    flow1.setName("开始节点 -> 任务节点1");
    flow1.setSourceRef("start");
    flow1.setTargetRef("userTask1");
    
    // 创建节点
    SequenceFlow flow2 = new SequenceFlow();
    flow2.setId("flow2");
    flow2.setName("任务节点1 -> 任务节点2");
    flow2.setSourceRef("userTask1");
    flow2.setTargetRef("userTask2");
    
    // 创建节点
    SequenceFlow flow3 = new SequenceFlow();
    flow3.setId("flow3");
    flow3.setName("任务节点2 -> 结束节点");
    flow3.setSourceRef("userTask2");
    flow3.setTargetRef("end");
    
    // 创建开始节点
    StartEvent startEvent = new StartEvent();
    startEvent.setId("start");
    startEvent.setName("开始节点");
    startEvent.setOutgoingFlows(Arrays.asList(flow1));
    
    UserTask userTask1 = new UserTask();
    userTask1.setId("userTask1");
    userTask1.setName("任务节点1");
    userTask1.setIncomingFlows(Arrays.asList(flow1));
    userTask1.setOutgoingFlows(Arrays.asList(flow2));
    
    UserTask userTask2 = new UserTask();
    userTask2.setId("userTask2");
    userTask2.setName("任务节点2");
    userTask2.setIncomingFlows(Arrays.asList(flow2));
    userTask2.setOutgoingFlows(Arrays.asList(flow3));
    
    EndEvent endEvent = new EndEvent();
    endEvent.setId("end");
    endEvent.setName("结束节点");
    endEvent.setIncomingFlows(Arrays.asList(flow3));
    
    BpmnModel model = new BpmnModel();
    // 创建流程
    org.activiti.bpmn.model.Process process = new Process();
    process.setId("process1");
    
    process.addFlowElement(startEvent);
    process.addFlowElement(flow1);
    process.addFlowElement(userTask1);
    process.addFlowElement(flow2);
    process.addFlowElement(userTask2);
    process.addFlowElement(flow3);
    process.addFlowElement(endEvent);
    
    // 添加流程
    model.addProcess(process);
    return model;
} //end createBpmnModel
```
### (10).BpmnModel转换流程文档
> BpmnModel是流程文档部署模块中非常重要的一个类,深入学习该类有助于快速了解Activiti中每一个元素及其可以定义的属性.并可以方便的获取流程引擎元素提供新特性.

```
// 将BpmnModel转换为:XML
@Test
public void testConverToXML(){
    BpmnModel model = createBpmnModel();
    BpmnXMLConverter converter = new BpmnXMLConverter();
    byte[] content = converter.convertToXML(model,"UTF-8");
    String xml = new String(content);
    System.out.println(xml);
}
```

> 生成文档后效果

```
<?xml version="1.0" encoding="UTF-8"?>
<definitions xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:activiti="http://activiti.org/bpmn" xmlns:bpmndi="http://www.omg.org/spec/BPMN/20100524/DI" xmlns:omgdc="http://www.omg.org/spec/DD/20100524/DC" xmlns:omgdi="http://www.omg.org/spec/DD/20100524/DI" typeLanguage="http://www.w3.org/2001/XMLSchema" expressionLanguage="http://www.w3.org/1999/XPath" targetNamespace="http://www.activiti.org/test">
  <process id="process1" isExecutable="true">
    <startEvent id="start" name="开始节点"></startEvent>
    <sequenceFlow id="flow1" name="开始节点 -&gt; 任务节点1" sourceRef="start" targetRef="userTask1"></sequenceFlow>
    <userTask id="userTask1" name="任务节点1"></userTask>
    <sequenceFlow id="flow2" name="任务节点1 -&gt; 任务节点2" sourceRef="userTask1" targetRef="userTask2"></sequenceFlow>
    <userTask id="userTask2" name="任务节点2"></userTask>
    <sequenceFlow id="flow3" name="任务节点2 -&gt; 结束节点" sourceRef="userTask2" targetRef="end"></sequenceFlow>
    <endEvent id="end" name="结束节点"></endEvent>
  </process>
  <bpmndi:BPMNDiagram id="BPMNDiagram_process1">
    <bpmndi:BPMNPlane bpmnElement="process1" id="BPMNPlane_process1"></bpmndi:BPMNPlane>
  </bpmndi:BPMNDiagram>
</definitions>
```
### (11).流程文档转换BpmnModel
```
@Test
public void testConverToBpmnModel() throws Exception {
    InputStream is = UserTaskTest.class.getClassLoader().getResource("user-task.bpmn").openStream();
    // InputStreamSource实现了InputStreamProvider.
    // 
    InputStreamSource inputStreamProvider = new InputStreamSource(is);
    BpmnXMLConverter converter = new BpmnXMLConverter();
    BpmnModel model = converter.convertToBpmnModel(inputStreamProvider, true, false, "UTF-8");
    Assert.assertTrue(null != model);
}
```
### (12).InputStreamProvider
```
// 资源提供者
public interface InputStreamProvider {
  InputStream getInputStream();
}
```