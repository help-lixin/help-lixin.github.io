---
layout: post
title: 'SpringCloud Gateway入门案例(二)'
date: 2020-12-15
author: 李新
tags: SpringCloudGateway
---

### (1). 环境准备工作
> 1. eureka注册中心(1111/2222).    
> 2. 服务提供者(TEST-PROVIDER:8080).   
> 3. 服务消费者(TEST-CONSUMER-FEIGN:7070).   
> 4. 项目之间的依赖:TEST-CONSUMER-FEIGN--> TEST-PROVIDER.    
> 5. 测试:curl http://localhost:7070/consumer

!["准备工作"](/assets/spring-cloud-gateway/imgs/spring-cloud-gateway-start.jpg)

### (2). Application
```
package help.lixin;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.SpringBootConfiguration;
import org.springframework.boot.autoconfigure.EnableAutoConfiguration;

@SpringBootConfiguration
@EnableAutoConfiguration
public class Application {
	public static void main(String[] args) {
		SpringApplication.run(Application.class, args);
	}
}
```
### (3). application.yml
```
#端口
server:
  port: 9000

# org.springframework.cloud.gateway.route.Route
# org.springframework.cloud.gateway.handler.predicate.PathRoutePredicateFactory
# id : 路由ID,需要做到唯一
# uri : 微服务的地址
# predicates : 断言(判断条件)
# 路由规则(Path): 匹配URL的请求,将匹配的请求追加在目标URI之后
# http://localhost:9000/consumer

spring:
  application:
    name: gateway-server  # 应用名称
  cloud:
    gateway:
      routes:
        - id: test-consumer-service
          uri: "http://localhost:7070/"
          predicates: 
            - Path=/consumer/**

```
### (4). pom.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>help.lixin</groupId>
	<artifactId>spring-colue-gateway</artifactId>
	<packaging>jar</packaging>
	<version>1.1.0</version>
	<name>spring-colue-gateway ${project.version}</name>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<maven.compiler.source>1.8</maven.compiler.source>
		<maven.compiler.target>1.8</maven.compiler.target>
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
		</dependencies>
	</dependencyManagement>

	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-gateway</artifactId>
		</dependency>
	</dependencies>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
				<configuration>
					<mainClass>help.lixin.Application</mainClass>
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
### (5). 访问代理服务器
> 目标:访问代理服务器(9000),它能正确解析到后面的微服上(7000).        
> http://localhost:9000/consumer

!["spring cloud gateway proxy consumer"](/assets/spring-cloud-gateway/imgs/spring-cloud-proxy-consumer-result.jpg)

