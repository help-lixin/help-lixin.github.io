---
layout: post
title: 'Spring WebFlux Hello World(Function)'
date: 2020-07-21
author: 李新
tags: Spring WebFlux
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

### (2).HelloHandler
```
package help.lixin.samples.handler;

import org.springframework.http.MediaType;
import org.springframework.web.reactive.function.BodyInserters;
import org.springframework.web.reactive.function.server.ServerRequest;
import org.springframework.web.reactive.function.server.ServerResponse;

import reactor.core.publisher.Mono;

public class HelloHandler {
	public Mono<ServerResponse> mono(ServerRequest request){
		// throw new RuntimeException("error");
		return ServerResponse.ok()    //
			    .contentType(MediaType.TEXT_PLAIN) //
			    .body(BodyInserters.fromObject("hello mono webflux"));
	}

	public Mono<ServerResponse> flux(ServerRequest request) {
		return ServerResponse.ok() //
				.contentType(MediaType.TEXT_PLAIN) //
				.body(BodyInserters.empty());
	}
	
}

```
### (3).RouteConfig
```
package help.lixin.samples.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.reactive.function.server.RequestPredicates;
import org.springframework.web.reactive.function.server.RouterFunction;
import org.springframework.web.reactive.function.server.RouterFunctions;
import org.springframework.web.reactive.function.server.ServerResponse;

import help.lixin.samples.handler.HelloHandler;

@Configuration
public class RouteConfig {

	@Bean
	public HelloHandler helloHandler() {
		return new HelloHandler();
	}

	@Bean
	public RouterFunction<ServerResponse> routes(HelloHandler helloHandler) {
		return RouterFunctions //
				.route(RequestPredicates.path("/api/mono"), helloHandler::mono);
	}
}

```
### (4).ErrorLogHandler
```
package help.lixin.samples.advice;

import java.io.Serializable;

import org.springframework.core.annotation.Order;
import org.springframework.core.io.buffer.DataBuffer;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import org.springframework.web.server.WebExceptionHandler;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;

import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

@Component
@Order(-2)
public class ErrorLogHandler implements WebExceptionHandler {

	@Override
	public Mono<Void> handle(ServerWebExchange exchange, Throwable ex) {
		exchange.getResponse().setStatusCode(HttpStatus.OK);
//		exchange.getResponse().getHeaders().add(HttpHeaders.CONTENT_TYPE, "application/json;charset=utf-8");
		ObjectMapper objectMapper = new ObjectMapper();
		
		Response response = new Response();
		response.setCode(HttpStatus.OK.value());
		response.setMessage("error ");
		byte[] bytes;
		DataBuffer wrap = null;
		try {
			bytes = objectMapper.writeValueAsBytes(response);
			wrap = exchange.getResponse().bufferFactory().wrap(bytes);
		} catch (JsonProcessingException ignore) {
		}
		return exchange.getResponse().writeWith(Flux.just(wrap));
	}
}

class Response implements Serializable {
	private static final long serialVersionUID = 8212733970119974320L;
	private Integer code;
	private String message;

	public Integer getCode() {
		return code;
	}

	public void setCode(Integer code) {
		this.code = code;
	}

	public String getMessage() {
		return message;
	}

	public void setMessage(String message) {
		this.message = message;
	}
}

```
### (5).Application
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
### (6).访问
!["Spring WebFlux Hello World(Function)"](/assets/webflux/images/webflux-helloworld.png)