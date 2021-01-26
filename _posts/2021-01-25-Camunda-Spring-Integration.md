---
layout: post
title: 'Camunda 与Spring集成'
date: 2021-01-25
author: 李新
tags: Camunda
---

### (1). 项目结构
```
camunda-spring-example
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
│       │               └── ProcessEngineTest.java
│       └── resources
│           ├── applicationContext.xml
│           ├── logback-test.xml
│           ├── logging.properties
│           └── testconfig.properties
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
### (3). applicationContext.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:activiti="http://www.activiti.org/schema/spring/components"
       xsi:schemaLocation="http://www.springframework.org/schema/beans   http://www.springframework.org/schema/beans/spring-beans.xsd">
    
    <bean id="dataSource" class="org.springframework.jdbc.datasource.TransactionAwareDataSourceProxy">
        <property name="targetDataSource">
        <!-- 这里也只是利用了Spring产生的DataSource,生产环境,建议换成专业的DataSource -->
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
        <property name="transactionManager" ref="transactionManager"/>
        <property name="databaseSchemaUpdate" value="true"/>
        <property name="jobExecutorActivate" value="false"/>
        <property name="dbMetricsReporterActivate" value="false" />
        <property name="telemetryReporterActivate" value="false" />
    </bean>

    <!-- using ManagedProcessEngineFactoryBean allows registering the ProcessEngine with the BpmPlatform -->
    <bean id="processEngine" class="org.camunda.bpm.engine.spring.container.ManagedProcessEngineFactoryBean">
        <property name="processEngineConfiguration" ref="processEngineConfiguration"/>
    </bean>

    <bean id="repositoryService" factory-bean="processEngine" factory-method="getRepositoryService"/>
    <bean id="runtimeService" factory-bean="processEngine" factory-method="getRuntimeService"/>
    <bean id="taskService" factory-bean="processEngine" factory-method="getTaskService"/>
    <bean id="historyService" factory-bean="processEngine" factory-method="getHistoryService"/>
    <bean id="managementService" factory-bean="processEngine" factory-method="getManagementService"/>
</beans>
```
### (4). pom.xml
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
