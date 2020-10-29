---
layout: post
title: 'Eureka集群搭建'
date: 2019-06-29
author: 李新
tags: Eureka
---

### (1).pom.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>help.lixin</groupId>
	<artifactId>spring-cloud-eureka</artifactId>
	<packaging>jar</packaging>
	<version>1.1.0</version>
	<name>spring-cloud-eureka ${project.version}</name>

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
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
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
### (2).Application
```
package help.lixin.samples;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@SpringBootApplication
// 激活EureakServer
@EnableEurekaServer
public class Application {
	public static void main(String[] args) {
		SpringApplication.run(Application.class, args);
	}
}
```
### (3).application.properties
```
spring.profiles.active=dev1
```
### (4).application-dev1.properties
```
server.port=1111
spring.application.name=eureka-server

#eureka.client.register-with-eureka=true
#eureka.client.fetch-registry=true
eureka.instance.hostname=eureka1
eureka.client.service-url.defaultZone=http://eureka2:2222/eureka/
```
### (5).application-dev2.properties
```
server.port=2222
spring.application.name=eureka-server

#eureka.client.register-with-eureka=true
#eureka.client.fetch-registry=true
eureka.instance.hostname=eureka2
eureka.client.service-url.defaultZone=http://eureka1:1111/eureka/

```
### (6).配置hosts(/etc/hosts),好像必须配置成hosts,不能配置成:lochaost
```
127.0.0.1 eureka1
127.0.0.1 eureka2
```
### (7).启动
```
java -jar target/spring-cloud-eureka-1.1.0.jar --spring.profiles.active=dev1
java -jar target/spring-cloud-eureka-1.1.0.jar --spring.profiles.active=dev2
```
### (8).查看集群信息
!["Eureka集群图示"](/assets/eureka/imgs/eureka-cluster.png)