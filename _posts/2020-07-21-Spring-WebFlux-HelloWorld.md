---
layout: post
title: 'Spring WebFlux Hello World'
date: 2020-07-21
author: 李新
tags: SpringWebFlux
---

### (1).pom.xml配置
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>help.lixin</groupId>
	<artifactId>springwebflux-demo</artifactId>
	<packaging>jar</packaging>
	<version>1.1.0</version>
	<name>springwebflux ${project.version}</name>

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
           <!-- 导入webflux -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-webflux</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
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
### (2).HelloController
```
package help.lixin.samples.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

@RestController
@RequestMapping("/api")
public class HelloController {
   
   // webflux要求,返回的数据类型必须是:Flux/Mono
	@GetMapping("/mono")
	public Mono<Object> mono() {
		return Mono.create(callback -> {
			System.out.println("创建Mono");
			callback.success("hello mono webflux");
		}).doOnSubscribe((onSubscribe) -> { // 当有订阅者来订阅发布者的时候,该方法会调用
			System.out.println(onSubscribe);
			System.out.println(Thread.currentThread().getName() + "  doOnSubscribe: " + onSubscribe);
		}).doOnNext(o -> { // 当订阅者收到数据时，改方法会调用
			System.out.println(Thread.currentThread().getName() + "  doOnNext: " + o);
		});
	}

	@GetMapping("/flux")
	public Flux<String> flux() {
		return Flux.just("hello", "world", "webflux");
	}
}
```
### (3).Application
```
package help.lixin.samples;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {
	public static void main(String[] args) {
		SpringApplication.run(Application.class, args);
	}
}
```
### (4).启动
```
2020-07-21 15:02:39.317  INFO 1140 --- [           main] help.lixin.samples.Application           : Starting Application on lixin-macbook.local with PID 1140 (/Users/lixin/Workspace/springwebflux-demo/target/classes started by lixin in /Users/lixin/Workspace/springwebflux-demo)
2020-07-21 15:02:39.334  INFO 1140 --- [           main] help.lixin.samples.Application           : The following profiles are active: dev
2020-07-21 15:02:41.233  WARN 1140 --- [           main] reactor.netty.tcp.TcpResources           : [http] resources will use the default LoopResources: DefaultLoopResources {prefix=reactor-http, daemon=true, selectCount=4, workerCount=4}
2020-07-21 15:02:41.241  WARN 1140 --- [           main] reactor.netty.tcp.TcpResources           : [http] resources will use the default ConnectionProvider: PooledConnectionProvider {name=http, poolFactory=reactor.netty.resources.ConnectionProvider$$Lambda$195/634638280@30331109}
## NettyWebServer webflux源码入口
2020-07-21 15:02:43.068  INFO 1140 --- [           main] o.s.b.web.embedded.netty.NettyWebServer  : Netty started on port(s): 7070
2020-07-21 15:02:43.082  INFO 1140 --- [           main] help.lixin.samples.Application           : Started Application in 4.878 seconds (JVM running for 6.145)
```
### (5). 访问
!["Spring WebFlux Hello World"](/assets/webflux/images/webflux-helloworld.png)
