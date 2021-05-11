---
layout: post
title: 'ShedLock 入门程序(二)'
date: 2021-05-11
author: 李新
tags:  ShedLock
---

### (1). 项目结构如下
```
lixin-macbook:Workspace lixin$ tree shedlock-example
shedlock-example
├── pom.xml
├── src
│   ├── main
│   │   ├── java
│   │   │   └── help
│   │   │       └── lixin
│   │   │           └── samples
│   │   │               ├── ProviderApplication.java
│   │   │               ├── config
│   │   │               │   └── ShedLockConfig.java
│   │   │               ├── controller
│   │   │               │   └── HelloController.java
│   │   │               └── tasks
│   │   │                   └── HelloWorldTask.java
│   │   └── resources
│   │       ├── application.properties
│   │       └── logback-spring.xml
│   └── test
│       ├── java
│       └── resources
└── target
```

### (2). 添加依赖(pom.xml)
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>help.lixin</groupId>
	<artifactId>shedlock-example</artifactId>
	<packaging>jar</packaging>
	<version>1.0.0</version>
	<name>shedlock-example</name>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<maven.compiler.source>8</maven.compiler.source>
		<maven.compiler.target>8</maven.compiler.target>
	</properties>

	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-dependencies</artifactId>
				<version>Greenwich.RELEASE</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
			<dependency>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-dependencies</artifactId>
				<version>2.1.0.RELEASE</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-netflix</artifactId>
				<version>2.1.1.RELEASE</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>

	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		
		<!-- DataSource初始化(DataSourceConfiguration) -->
		<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jdbc</artifactId>
        </dependency>
        <dependency>
		  <groupId>mysql</groupId>
		  <artifactId>mysql-connector-java</artifactId>
		  <version>5.1.49</version>
		</dependency> 
        
        <!-- 添加ShedLock依赖 -->
		<dependency>
		    <groupId>net.javacrumbs.shedlock</groupId>
		    <artifactId>shedlock-spring</artifactId>
		    <version>4.23.1-SNAPSHOT</version>
		</dependency>
		<!-- 添加lock提供者 --> 
		<dependency>
            <groupId>net.javacrumbs.shedlock</groupId>
            <artifactId>shedlock-provider-jdbc-template</artifactId>
            <version>4.23.1-SNAPSHOT</version>
        </dependency>
        
        <dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>
	
</project>
```

### (3). application.properties
```
server.port=8080
spring.application.name=shedlock-example

# 连接池类型
spring.datasource.type=com.zaxxer.hikari.HikariDataSource
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.username=root
spring.datasource.password=123456
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/shedlock?useSSL=false
```
### (4). ShedLockConfig
```
package help.lixin.samples.config;

import javax.sql.DataSource;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.jdbc.core.JdbcTemplate;

import net.javacrumbs.shedlock.core.LockProvider;
import net.javacrumbs.shedlock.provider.jdbctemplate.JdbcTemplateLockProvider;

@Configuration
public class ShedLockConfig {
	
	// 配置锁的提供者
	@Bean
	public LockProvider lockProvider(@Autowired DataSource dataSource) {
        return new JdbcTemplateLockProvider(
            JdbcTemplateLockProvider.Configuration.builder()
            .withJdbcTemplate(new JdbcTemplate(dataSource))
            .usingDbTime() // Works on Postgres, MySQL, MariaDb, MS SQL, Oracle, DB2, HSQL and H2
            .build()
        );
	}
}

```
### (5). HelloWorldTask
```
package help.lixin.samples.tasks;

import java.util.Date;

import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

import net.javacrumbs.shedlock.spring.annotation.SchedulerLock;

@Component
public class HelloWorldTask {

	@Scheduled(cron = "*/5 * * * * ?")
	// ***********************************************
	// 定义锁的名称,最小锁住时间(lockAtLeastFor)和最大锁住时间(lockAtMostFor).
	// ***********************************************
	@SchedulerLock(name = "hello-world", lockAtLeastFor = "20000", lockAtMostFor = "30000")
	public void helloworld() {
		System.out.println(String.format("[%s] Hello World job run...", new Date()));
	}
}

```
### (6). ProviderApplication
```
package help.lixin.samples;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableScheduling;

import net.javacrumbs.shedlock.spring.annotation.EnableSchedulerLock;

@SpringBootApplication
@EnableScheduling
// ***************************************************
// 注意:需要启用注解@EnableSchedulerLock
// ***************************************************
@EnableSchedulerLock(defaultLockAtMostFor = "120m")
public class ProviderApplication {
	public static void main(String[] args) throws Exception {
		SpringApplication.run(ProviderApplication.class, args);
	}
}
```
### (7). 运行查看控制台
```
# 1. 测试不添加分布式锁(不加注解@SchedulerLock)的情况下(每5秒会执行一次).
[Tue May 11 15:49:45 CST 2021] Hello World job run...
[Tue May 11 15:49:50 CST 2021] Hello World job run...
[Tue May 11 15:49:55 CST 2021] Hello World job run...
[Tue May 11 15:50:00 CST 2021] Hello World job run...


# 2. 测试添加分布锁(增加注解@SchedulerLock)的情况下
#    虽然,定时任务是每隔5秒执行一次,但是,分布式锁定义的是:每次任务要锁住20秒.
#    所以,定时任务加锁失败的情况下,是直接跳过的.
[Tue May 11 15:51:35 CST 2021] Hello World job run...
[Tue May 11 15:52:00 CST 2021] Hello World job run...
[Tue May 11 15:52:25 CST 2021] Hello World job run...
[Tue May 11 15:52:45 CST 2021] Hello World job run...
```
### (8). 总结
> 有人肯定会想问:明明是个分布式锁,非要跟定时任务绑在一起,这个框架,能不能做到,不和定时任务绑定呢?  
> 答案是:可以的,因为:shedlock-spring是对shedlock-core的二次封装,以适应于定时任务场景而已.后面的小节,我会对ShedLock源码剖析.