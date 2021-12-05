---
layout: post
title: 'Spring Cloud Stream之Hello World(RocketMQ)'
date: 2020-07-03
author: 李新
tags: SpringCloudStream
---

### (1).启动RocketMQ
```
cd /Users/lixin/GitRepository/rocketmq-all-4.7.1/distribution/target/rocketmq-4.7.1/rocketmq-4.7.1/bin

// 启动nameserver
./mqnamesrv 

// 启动brokder
./mqbroker -n localhost:9876

// 创建主题
./mqadmin updateTopic -n localhost:9876 -c DefaultCluster -t test-topic
```
### (2).依赖添加 
> Spring Alibaba与Spring cloud选择:
https://github.com/alibaba/spring-cloud-alibaba/wiki/%E7%89%88%E6%9C%AC%E8%AF%B4%E6%98%8E

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>help.lixin</groupId>
	<artifactId>spring-colue-samples-stream</artifactId>
	<packaging>jar</packaging>
	<version>1.1.0</version>
	<name>spring-cloud-samples-stream ${project.version}</name>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<java.version>1.8</java.version>
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
                   <!-- spring cloud alibaba dependenices -->
			<dependency>
				<groupId>com.alibaba.cloud</groupId>
				<artifactId>spring-cloud-alibaba-dependencies</artifactId>
				<version>2.1.2.RELEASE</version>
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
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-actuator</artifactId>
		</dependency>
  
           <!-- stream -->
		<dependency>
			<groupId>com.alibaba.cloud</groupId>
			<artifactId>spring-cloud-starter-stream-rocketmq</artifactId>
		</dependency>
	</dependencies>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
				<configuration>
					<mainClass>help.lixin.samples.Application</mainClass>
				</configuration>
				<executions>
					<execution>
						<goals>
							<goal>repackage</goal>
						</goals>
					</execution>
				</executions>
			</plugin>
		</plugins>
	</build>
</project>
```
### (3).spring-cloud-sample-stream-provider(application-dev.properties)
```
## application-dev.properties
server.port=6060
spring.application.name=stream-provider

#stream
## defin nameserver-address
spring.cloud.stream.rocketmq.binder.namesrv-addr=127.0.0.1:9876
## defin name = output
spring.cloud.stream.bindings.output.destination=test-topic
```
### (4).StreamSourceApplication
```
package help.lixin.samples;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.stream.annotation.EnableBinding;
import org.springframework.cloud.stream.messaging.Source;

import help.lixin.samples.service.ProvderService;

@SpringBootApplication
// 启用Source,会自动注册
@EnableBinding({ Source.class })
public class StreamSourceApplication implements CommandLineRunner {
	
	@Autowired
	private ProvderService provderService;
	
	public static void main(String[] args) throws Exception {
		SpringApplication.run(StreamSourceApplication.class, args);
	}

	public void run(String... args) throws Exception {
		provderService.sned("lixin hello world!!!!!");
	}
}
```

### (5).ProvderService
```
package help.lixin.samples.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.messaging.Source;
import org.springframework.integration.support.MessageBuilder;
import org.springframework.stereotype.Component;

@Component
public class ProvderService {
	
	@Autowired
	private Source source;
	
	public void sned(String message) {
		source.output().send(MessageBuilder.withPayload(message).build());
	}
}
```

### (6).spring-cloud-sample-stream-consumer(application-dev.properties)
```
## application-dev.properties
server.port=6061
spring.application.name=stream-consumer

#stream
spring.cloud.stream.rocketmq.binder.namesrv-addr=127.0.0.1:9876
# 定义名称为:input与test-topic关系
spring.cloud.stream.bindings.input.destination=test-topic
spring.cloud.stream.bindings.input.group=${spring.application.name}
```

### (7).StreamSinkApplication
```
## StreamSinkApplication
package help.lixin.samples;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.stream.annotation.EnableBinding;
import org.springframework.cloud.stream.messaging.Sink;

@SpringBootApplication
@EnableBinding({ Sink.class})
public class StreamSinkApplication {
	public static void main(String[] args) throws Exception {
		SpringApplication.run(StreamSinkApplication.class, args);
	}
}
```

### (8).ConsumerService
```
package help.lixin.samples.service;

import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.cloud.stream.messaging.Sink;
import org.springframework.stereotype.Component;

@Component
public class ConsumerService {

   // 接收消息
	@StreamListener(Sink.INPUT)
	public void sned(String message) {
		System.out.println("revice message: " + message);
	}
}
```
### (9). Spring Cloud Stream配置类
```
# BindingServiceProperties是Spring Cloud Stream的配置类
org.springframework.cloud.stream.config.BindingServiceProperties


# RocketMQ针对Spring Cloud Stream的扩展配置类
com.alibaba.cloud.stream.binder.rocketmq.properties.RocketMQExtendedBindingProperties
com.alibaba.cloud.stream.binder.rocketmq.properties.RocketMQBinderConfigurationProperties
```
### (10).总结
> 原理看另一篇