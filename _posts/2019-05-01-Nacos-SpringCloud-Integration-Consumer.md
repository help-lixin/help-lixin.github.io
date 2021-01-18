---
layout: post
title: 'Nacos与Spring Cloud集成-消费者(五)'
date: 2019-05-01
author: 李新
tags: Nacos SpringCloud
---

### (1). spring-cloud-nacos-consumer项目结构
```
spring-cloud-nacos-consumer
├── pom.xml
├── src
│   ├── main
│   │   ├── java
│   │   │   └── help
│   │   │       └── lixin
│   │   │           └── samples
│   │   │               ├── ConsumerApplication.java
│   │   │               └── controller
│   │   │                   └── ConsumerController.java
│   │   └── resources
│   │       └── bootstrap.properties
│   └── test
│       ├── java
│       └── resources
└── target
```
### (2). pom.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>help.lixin</groupId>
	<artifactId>spring-colue-samples-consumer</artifactId>
	<packaging>jar</packaging>
	<version>1.1.0</version>
	<name>spring-colue-samples-consumer ${project.version}</name>

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
		
		<dependency>
		    <groupId>com.alibaba.cloud</groupId>
		    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
		</dependency>
		<!-- nacos 服务发现组件 -->
		<dependency>
		    <groupId>com.alibaba.cloud</groupId>
		    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
		</dependency>
		<!--  引入ribbon  -->
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
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
### (3). bootstrap.properties
```
server.port=7070
spring.application.name=test-consumer

# nacos配置地址
spring.cloud.nacos.config.server-addr=127.0.0.1:8848
# 配置内容的数据格式
spring.cloud.nacos.config.file-extension=properties
# 指定namespace
spring.cloud.nacos.config.namespace=45a6112b-a866-4e92-a8d6-9e440bcdebbe
# 指定group
spring.cloud.nacos.config.group=erp:test-consumer

# 注册中心地址
spring.cloud.nacos.discovery.server-addr=127.0.0.1:8848
# 指定namespace
spring.cloud.nacos.discovery.namespace=45a6112b-a866-4e92-a8d6-9e440bcdebbe
# 指定group(不建议配置,或者:统一好一个group)
#spring.cloud.nacos.discovery.group=erp:test-provider
```
### (4). ConsumerController
```
package help.lixin.samples.controller;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;

@RestController
@RefreshScope
public class ConsumerController {
	private Logger logger = LoggerFactory.getLogger(ConsumerController.class);

	@Autowired
	private RestTemplate restTemplate;

	@Value("${remoteSwitch:true}")
	private boolean remoteSwitch;

	@Value("${showMsg:local consumer...}")
	private String showMsg;

	@GetMapping("/consumer")
	public String index() {
		logger.debug("consumer....");
		String url = "http://test-provider/hello";
		String result = restTemplate.getForEntity(url, String.class).getBody();
		return showMsg + result;
	}
}
```
### (5). ConsumerApplication
```
package help.lixin.samples;

import java.util.ArrayList;
import java.util.List;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.context.annotation.Bean;
import org.springframework.http.client.ClientHttpRequestInterceptor;
import org.springframework.http.client.support.BasicAuthenticationInterceptor;
import org.springframework.web.client.RestTemplate;

@EnableDiscoveryClient
@SpringBootApplication
public class ConsumerApplication {

	@Bean
	@LoadBalanced
	RestTemplate restTemplate() {
		ClientHttpRequestInterceptor basicAuthenticationInterceptor = new BasicAuthenticationInterceptor("lixin", "123456");
		List<ClientHttpRequestInterceptor> interceptors = new ArrayList<ClientHttpRequestInterceptor>();
		interceptors.add(basicAuthenticationInterceptor);
		RestTemplate restTemplate = new RestTemplate();
		restTemplate.setInterceptors(interceptors);
		return restTemplate;
	}

	public static void main(String[] args) {
		SpringApplication.run(ConsumerApplication.class, args);
	}
}
```
### (6). 项目源码下载
["spring-cloud-nacos-consumer.zip"](/assets/nacos/spring-cloud-nacos-consumer.zip)
### (7). 启动项目,查看服务列表

!["Nacos 服务提供者列表"](/assets/nacos/imgs/nacos-provider-consumer-list.jpg)