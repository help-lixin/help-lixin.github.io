---
layout: post
title: 'Camunda 与Spring集成并测试'
date: 2021-01-25
author: 李新
tags: Camunda
---

### (1). 项目结构
```
camunda-spring-example/
├── pom.xml
├── src
│   ├── main
│   │   ├── java
│   │   └── resources
│   └── test
│       ├── java
│       │   └── help
│       │       └── lixin
│       │           └── camunda
│       │               ├── RuntimeServiceTest.java
│       │               └── ProcessEngineTest.java
│       └── resources
│           ├── application.properties
│           ├── applicationContext.xml
│           ├── logback-spring.xml
└── target
```
### (2). ProcessEngineTest
```
package help.lixin.camunda;

import org.camunda.bpm.engine.HistoryService;
import org.camunda.bpm.engine.RepositoryService;
import org.camunda.bpm.engine.RuntimeService;
import org.camunda.bpm.engine.TaskService;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class ProcessEngineTest {
	public static void main(String[] args) {
		ApplicationContext ctx = new ClassPathXmlApplicationContext("applicationContext.xml");
		RepositoryService repositoryService = ctx.getBean(RepositoryService.class);
		RuntimeService runtimeService = ctx.getBean(RuntimeService.class);
		TaskService taskService = ctx.getBean(TaskService.class);
		HistoryService historyService = ctx.getBean(HistoryService.class);
		System.out.println(repositoryService);
		System.out.println(runtimeService);
		System.out.println(taskService);
		System.out.println(historyService);
	}
}
```

### (3). RuntimeServiceTest
```
package help.lixin.camunda;

import java.util.Arrays;
import java.util.Date;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

import org.camunda.bpm.engine.IdentityService;
import org.camunda.bpm.engine.RepositoryService;
import org.camunda.bpm.engine.RuntimeService;
import org.camunda.bpm.engine.TaskService;
import org.camunda.bpm.engine.identity.Group;
import org.camunda.bpm.engine.identity.Tenant;
import org.camunda.bpm.engine.impl.identity.Authentication;
import org.camunda.bpm.engine.impl.persistence.entity.GroupEntity;
import org.camunda.bpm.engine.impl.persistence.entity.TenantEntity;
import org.camunda.bpm.engine.impl.persistence.entity.UserEntity;
import org.camunda.bpm.engine.runtime.ProcessInstance;
import org.camunda.bpm.engine.runtime.ProcessInstantiationBuilder;
import org.camunda.bpm.engine.task.Task;
import org.junit.BeforeClass;
import org.junit.Test;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class RuntimeServiceTest {

	static final String GROUP_ID = "7305184575594516AEC5A992F539A074";
	static final String USER_ID_ZHANGSAN = "zhangsan";
	static final String USER_ID_LISHI = "lishi";
	static final String USER_ID_WANGWU = "wangwu";
	static final String USER_ID_ZHAOLIU = "zhaoliu";
	static final String TENANT_ID = "4922021F02E248BF84CF3DB4AD0AD694";

	static ApplicationContext ctx = null;

	@BeforeClass
	public static void before() {
		ctx = new ClassPathXmlApplicationContext("applicationContext.xml");
	}

	@Test
	public void testInitData() {
		IdentityService identityService = ctx.getBean(IdentityService.class);

		// 添加租户
		Tenant tenant = new TenantEntity();
		tenant.setId(TENANT_ID);
		tenant.setName("XXX科技有限公司");
		identityService.saveTenant(tenant);

		// 添加Group
		Group group = new GroupEntity();
		group.setId(GROUP_ID);
		group.setName("研发部门");
		group.setType("DEVELOPER");
		identityService.saveGroup(group);

		// 组和租户ID进行绑定
		identityService.createTenantGroupMembership(TENANT_ID, GROUP_ID);

		// zhangsan
		UserEntity zhangsan = new UserEntity();
		// id即是登录的账号
		zhangsan.setId("zhangsan");
		zhangsan.setPassword("1");
		zhangsan.setSalt("1");
		zhangsan.setEmail("zhangsan@126.com");
		zhangsan.setFirstName("zhang");
		zhangsan.setLastName("san");

		// lishi
		UserEntity lishi = new UserEntity();
		// id即是登录的账号
		lishi.setId("lishi");
		lishi.setPassword("1");
		lishi.setSalt("1");
		lishi.setEmail("lishi@126.com");
		lishi.setFirstName("li");
		lishi.setLastName("shi");

		// wangwu
		UserEntity wangwu = new UserEntity();
		// id即是登录的账号
		wangwu.setId("wangwu");
		wangwu.setPassword("1");
		wangwu.setSalt("1");
		wangwu.setEmail("wangwu@126.com");
		wangwu.setFirstName("wang");
		wangwu.setLastName("wu");

		// zhaoliu
		UserEntity zhaoliu = new UserEntity();
		// id即是登录的账号
		zhaoliu.setId("zhaoliu");
		zhaoliu.setPassword("1");
		zhaoliu.setSalt("1");
		zhaoliu.setEmail("zhaoliu@126.com");
		zhaoliu.setFirstName("zhao");
		zhaoliu.setLastName("liu");

		identityService.saveUser(zhangsan);
		identityService.saveUser(lishi);
		identityService.saveUser(wangwu);
		identityService.saveUser(zhaoliu);

		// 用户与组绑定关系
		identityService.createMembership(zhangsan.getId(), GROUP_ID);
		identityService.createMembership(lishi.getId(), GROUP_ID);
		identityService.createMembership(wangwu.getId(), GROUP_ID);
		identityService.createMembership(zhaoliu.getId(), GROUP_ID);

		// 租户与用户绑定关系
		identityService.createTenantUserMembership(TENANT_ID, zhangsan.getId());
		identityService.createTenantUserMembership(TENANT_ID, lishi.getId());
		identityService.createTenantUserMembership(TENANT_ID, wangwu.getId());
		identityService.createTenantUserMembership(TENANT_ID, zhaoliu.getId());
	}

	/**
	 * 部署流程定义
	 */
	@Test
	public void testProcessDeploy() {
		// ACT_RE_DEPLOYMENT : 流程部署.
		// insert into ACT_RE_DEPLOYMENT(ID_, NAME_, DEPLOY_TIME_, SOURCE_, TENANT_ID_)
		// values(?, ?, ?, ?, ?)

		// ACT_GE_BYTEARRAY : 流程定义二进制数据.
		// insert into ACT_GE_BYTEARRAY( ID_, NAME_, BYTES_, DEPLOYMENT_ID_, GENERATED_,
		// TENANT_ID_, TYPE_, CREATE_TIME_, REV_) values ( ?, ?, ?, ?, ?, ?, ?, ?, 1)

		// ACT_RE_PROCDEF : 流程定义
		// insert into ACT_RE_PROCDEF(ID_, CATEGORY_, NAME_, KEY_, VERSION_,
		// DEPLOYMENT_ID_, RESOURCE_NAME_, DGRM_RESOURCE_NAME_, HAS_START_FORM_KEY_,
		// SUSPENSION_STATE_, TENANT_ID_, VERSION_TAG_, HISTORY_TTL_, STARTABLE_, REV_)
		// values (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, 1 )
		RepositoryService repositoryService = ctx.getBean(RepositoryService.class);
		// 部署流程定义
		repositoryService.createDeployment() //
				.name("请假流程") //
				.source("Test") //
				.tenantId(TENANT_ID) //
				.enableDuplicateFiltering() //
				.addClasspathResource("leave.bpmn") //
				.deploy();
	}

	
	@Test
	public void testStartProcessInstanceByKey() {
		// 流程定义 : 通过Camunda Modeler设计出来的流程定义信息.
		// 流程实例 : 从流程定义(模板),创建出来的流程实例.
		// 执行实例 : 一个流程实例包括所有的运行节点(执行实例).可通过流程实例对象来了解当前流程进度等信息.
		// 在流程运转的过程中,永远执行的是自己对应的执行实例.
		RuntimeService runtimeService = ctx.getBean(RuntimeService.class);
		IdentityService identityService = ctx.getBean(IdentityService.class);
		// 对应流程定义中声明的:id
		String processDefinitionKey = "leave";
		Map<String,Object> vars = new HashMap<String,Object>();
		vars.put("day", 4L);
		vars.put("start_time", new Date());
		vars.put("end_time", new Date());
		vars.put("reason", "go home...");
		
		// 在有租户的情况下,还真只能这样写.否则会报错(也可以禁用租户检查).
		Authentication authentication = new Authentication(USER_ID_ZHANGSAN, Arrays.asList(GROUP_ID), Arrays.asList(TENANT_ID));
		// 把当前用户信息,设置到线程上下文中.
		identityService.setAuthentication(authentication);
		try {
			ProcessInstantiationBuilder processInstantiationBuilder = runtimeService.createProcessInstanceByKey(processDefinitionKey);
			processInstantiationBuilder.setVariables(vars);
			processInstantiationBuilder.processDefinitionTenantId(TENANT_ID);
			ProcessInstance processInstance = processInstantiationBuilder.execute();
			// 不能再这样启动流程了(因为:tenant_id没地方传递进去)
			// ProcessInstance processInstance = runtimeService.startProcessInstanceByKey(processDefinitionKey,vars);
			String format = "id: " + processInstance.getId() + "  businessKey: " + processInstance.getBusinessKey()
					+ " tenantId: " + processInstance.getTenantId() + " processDefinitionId: "
					+ processInstance.getProcessDefinitionId();
			System.out.println(format);
		}finally {
			// 记得要清除当前用户信息
			identityService.clearAuthentication();
		}
	}

	/**
	 * 查询待办任务
	 */
	@Test
	public void testQueryTask() {
//		 select distinct RES.REV_, RES.ID_, RES.NAME_, RES.PARENT_TASK_ID_, RES.DESCRIPTION_, RES.PRIORITY_, RES.CREATE_TIME_, RES.OWNER_, RES.ASSIGNEE_, RES.DELEGATION_, RES.EXECUTION_ID_, RES.PROC_INST_ID_, RES.PROC_DEF_ID_, RES.CASE_EXECUTION_ID_, RES.CASE_INST_ID_, RES.CASE_DEF_ID_, RES.TASK_DEF_KEY_, RES.DUE_DATE_, RES.FOLLOW_UP_DATE_, RES.SUSPENSION_STATE_, RES.TENANT_ID_ from ACT_RU_TASK RES WHERE ( 1 = 1 and RES.ASSIGNEE_ = ? ) order by RES.ID_ asc LIMIT ? OFFSET ? 
//       Parameters: zhangsan(String), 2147483647(Integer), 0(Integer)
		
		TaskService taskService = ctx.getBean(TaskService.class);
		List<Task> tasks = taskService.createTaskQuery() //
				.taskAssignee(USER_ID_ZHANGSAN).list();
//				.taskAssignee(USER_ID_ZHAOLIU).list();
		for (Task task : tasks) {
			// task.getTaskDefinitionKey() : 流程定义KEY
			// task.getId()                : 执行任务ID
			System.out.println("id: " + task.getId() + " key: " + task.getTaskDefinitionKey());
		}
	}
	
	
	@Test
	public void testComleteTask() {
		TaskService taskService = ctx.getBean(TaskService.class);
		// 获得最后一条数据
		String taskId = "1413";
		taskService.complete(taskId);
		// 可传递变量
		// taskService.complete(taskId, variables);
	}
	
	@Test
	public void testDeleteProcess() {
		// 删除指定的实例
		RuntimeService runtimeService = ctx.getBean(RuntimeService.class);
		runtimeService.deleteProcessInstance("501", "delete test");
		runtimeService.deleteProcessInstance("701", "delete test");
	}
}
```
### (4). applicationContext.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:activiti="http://www.activiti.org/schema/spring/components"
       xsi:schemaLocation="http://www.springframework.org/schema/beans   http://www.springframework.org/schema/beans/spring-beans.xsd">
    
    <bean id="dataSource" class="org.springframework.jdbc.datasource.TransactionAwareDataSourceProxy">
        <property name="targetDataSource">
        <!-- 这里也只是利用了Spring产生的DataSource -->
            <bean class="org.springframework.jdbc.datasource.SimpleDriverDataSource">
                <property name="driverClass" value="com.mysql.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://127.0.0.1:3306/camunda?useUnicode=true"/>
                <property name="username" value="root"/>
                <property name="password" value="123456"/>
            </bean>
        </property>
    </bean>

    <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource"/>
    </bean>

    <bean id="processEngineConfiguration" class="org.camunda.bpm.engine.spring.SpringProcessEngineConfiguration">
        <property name="dataSource" ref="dataSource"/>
        <property name="history" value="full"/>
        <property name="transactionManager" ref="transactionManager"/>
        <property name="databaseSchemaUpdate" value="false"/>
        <!-- <property name="databaseSchemaUpdate" value="create-drop"/> -->
        <property name="jobExecutorActivate" value="false"/>
        <property name="dbMetricsReporterActivate" value="false" />
        <property name="telemetryReporterActivate" value="false" />
    </bean>

    <!-- using ManagedProcessEngineFactoryBean allows registering the ProcessEngine with the BpmPlatform -->
    <bean id="processEngine" class="org.camunda.bpm.engine.spring.container.ManagedProcessEngineFactoryBean">
        <property name="processEngineConfiguration" ref="processEngineConfiguration"/>
    </bean>
	
	<bean id="formService" factory-bean="processEngine" factory-method="getFormService"/>
	<bean id="externalTaskService" factory-bean="processEngine" factory-method="getExternalTaskService"/>
	<bean id="authorizationService" factory-bean="processEngine" factory-method="getAuthorizationService"/>
	<bean id="identityService" factory-bean="processEngine" factory-method="getIdentityService"/>
    <bean id="repositoryService" factory-bean="processEngine" factory-method="getRepositoryService"/>
    <bean id="runtimeService" factory-bean="processEngine" factory-method="getRuntimeService"/>
    <bean id="taskService" factory-bean="processEngine" factory-method="getTaskService"/>
    <bean id="historyService" factory-bean="processEngine" factory-method="getHistoryService"/>
    <bean id="managementService" factory-bean="processEngine" factory-method="getManagementService"/>
</beans>
```

### (5). application.properties
```
logging.config=classpath:logback-spring.xml
```

### (6). logback-spring.xml
```
<configuration>
	<appender name="STDOUT"
		class="ch.qos.logback.core.ConsoleAppender">
		<encoder>
			<pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n
			</pattern>
		</encoder>
	</appender>
	<logger name="org.apache.ibatis" level="debug" />
	<logger name="javax.activation" level="info" />
	<logger name="org.springframework" level="info" />
	<logger name="org.mortbay.log" level="info" />
	<logger name="org.camunda" level="info" />
	<logger name="org.camunda.bpm.engine.test" level="debug" />
	<root level="all">
		<appender-ref ref="STDOUT" />
	</root>
</configuration>
```
### (7). pom.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>help.lixin.camunda</groupId>
	<artifactId>camunda-spring-example</artifactId>
	<packaging>jar</packaging>
	<version>1.1.0</version>
	<name>camunda-spring-example-${project.version}</name>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<maven.compiler.source>1.8</maven.compiler.source>
		<maven.compiler.target>1.8</maven.compiler.target>
		<java.version>1.8</java.version>
		<spring.version>5.2.8.RELEASE</spring.version>
	</properties>

	<dependencies>
		<dependency>
			<groupId>org.camunda.bpm</groupId>
			<artifactId>camunda-engine-spring</artifactId>
			<version>7.14.0</version>
		</dependency>
		<dependency>
			<groupId>mysql</groupId>
			<artifactId>mysql-connector-java</artifactId>
			<version>5.1.49</version>
		</dependency>
		<dependency>  
            <groupId>org.springframework</groupId>  
            <artifactId>spring-core</artifactId>  
            <version>${spring.version}</version>  
        </dependency>  
        <dependency>  
            <groupId>org.springframework</groupId>  
            <artifactId>spring-beans</artifactId>  
            <version>${spring.version}</version>  
        </dependency>  
        <dependency>  
            <groupId>org.springframework</groupId>  
            <artifactId>spring-context</artifactId>  
            <version>${spring.version}</version>  
        </dependency>
        <dependency>  
            <groupId>org.springframework</groupId>  
            <artifactId>spring-jdbc</artifactId>  
            <version>${spring.version}</version>  
        </dependency> 
		<dependency>  
            <groupId>org.springframework</groupId>  
            <artifactId>spring-tx</artifactId>
            <version>${spring.version}</version>  
        </dependency> 
		<dependency>
			<groupId>ch.qos.logback</groupId>
			<artifactId>logback-classic</artifactId>
			<version>1.2.3</version>
			<scope>test</scope>
		</dependency>
		<dependency>
			<groupId>junit</groupId>
			<artifactId>junit</artifactId>
			<version>4.12</version>
			<scope>test</scope>
		</dependency>
	</dependencies>
</project>
```
