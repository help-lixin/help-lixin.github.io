---
layout: post
title: 'SpringCloud @RefreshScope(实现配置自动刷新)'
date: 2018-05-01
author: 李新
tags: SpringCloud
---

### (1). 需求
> 在微服务越来越流行的当下,运维要配置的内容也越来越多,期望:当配置更新后应用的配置也要热更新.有三种解决方案:  
> 1. 方案一:在Bean创建时,记录@Value信息,然后通过反射来更新.   
> 2. 方案二:把Spring中的Bean销毁掉,下次使用Bean时,Spring会自动创建这个Bean的实例.  
> 3. 方案三:如果Bean上有@Value注解,则,为Bean进行代码增强(比如:实现某个接口,当有配置更新时,这个接口能接受,并反射更新Field的值).  
> 4. 我以前所在的公司,使用的是方案一.SpringCloud使用的是方案二,在这里,研究下方案二.

### (2). 测试步骤
> 1. 添加依赖.  
> 2. 为要刷新的Bean添加注解(@RefreshScope).    
> 3. 模拟触发刷新.   

### (3). 添加依赖
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>help.lixin</groupId>
	<artifactId>refresh-bean-example</artifactId>
	<packaging>jar</packaging>
	<version>1.1.0</version>
	<name>refresh-bean-example ${project.version}</name>

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
		<!-- 重点依赖 -->
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>
</project>
```
### (4). 为Bean添加注解(@RefreshScope)
```
package help.lixin.refresh.bean.controller;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
// ******************************************
// 这一步很重要.
// ******************************************
@RefreshScope
public class HelloController {
	private Logger logger = LoggerFactory.getLogger(HelloController.class);

	// 读取配置文件
	@Value("${help.lixin.zipkin.controller.HelloController.name}")
	private String name;

	@GetMapping("/hello")
	public String hello() {
		logger.info("hello world!!!" + name);
		return "Hello World!!! " + name + " -- " + this + "\r\n";
	}
}
```
### (5). 模拟触发刷新(RefreshController).
```
package help.lixin.refresh.bean.controller;

import java.util.UUID;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.cloud.context.scope.refresh.RefreshScope;
import org.springframework.context.ApplicationContext;
import org.springframework.core.env.MapPropertySource;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class RefreshController {
	private Logger logger = LoggerFactory.getLogger(RefreshController.class);

	//Spring Cloud提供了刷新接口
	@Autowired
	private RefreshScope refreshScope;
	
	
	@Autowired
	@Qualifier("mapPropertySource")
	private MapPropertySource mapPropertySource;

	@GetMapping("/refresh")
	public String refresh() {
		logger.debug("start refresh");

		// *********************************************
		// 更新配置内容
		// 这里的:MapPropertySource可以理解为分布式框架,在获取到配置后,存储的容器.
		// *********************************************
		String key = "help.lixin.zipkin.controller.HelloController.name";
		String value = "lixin + " + UUID.randomUUID().toString();
		mapPropertySource.getSource().put(key, value);

		// 刷新整个Spring容器
		// refreshScope.refreshAll();
		// 刷新单个bean
		refreshScope.refresh("helloController");
		return "SUCCESS " + this + " \r\n";
	}
}
```
### (6). RefreshBeanConfig
```
package help.lixin.refresh.bean.config;

import java.util.HashMap;
import java.util.Map;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.env.MapPropertySource;

import help.lixin.refresh.bean.processor.EnvironmentExt;

@Configuration
public class RefreshBeanConfig {

	// 配置文件载体
	@Bean
	public MapPropertySource mapPropertySource() {
		String name = "default";
		Map<String, Object> source = new HashMap<String, Object>();
		MapPropertySource mapPropertySource = new MapPropertySource(name, source);
		return mapPropertySource;
	}

	// 
	@Bean
	public EnvironmentExt customEnvironmentPostProcessor(MapPropertySource mapPropertySource) {
		return new EnvironmentExt(mapPropertySource);
	}
}

```

### (7). EnvironmentExt
```
package help.lixin.refresh.bean.processor;

import org.springframework.context.EnvironmentAware;
import org.springframework.core.env.ConfigurableEnvironment;
import org.springframework.core.env.Environment;
import org.springframework.core.env.PropertySource;

public class EnvironmentExt implements EnvironmentAware {
	private PropertySource<?> propertySource;

	private Environment environment;

	public EnvironmentExt(PropertySource<?> propertySource) {
		this.propertySource = propertySource;
	}

	public PropertySource<?> getPropertySource() {
		return propertySource;
	}

	public Environment getEnvironment() {
		return environment;
	}

	public void setEnvironment(Environment environment) {
		this.environment = environment;
		if (environment instanceof ConfigurableEnvironment) {
			// ********************************************************
			// env.getPropertySources()返回的对象是:MutablePropertySources
			// MutablePropertySources.propertySourceList类型为:CopyOnWriteArrayList.是一个不可变的对象
			// 但是,可以Holder住:PropertySource,改变:PropertySource的内容即可.
			// ********************************************************
			ConfigurableEnvironment env = (ConfigurableEnvironment) environment;
			env.getPropertySources().addFirst(propertySource);
		}
	}
}

```
### (8). application.properties
```
server.port=8080
spring.application.name=test-provider
help.lixin.zipkin.controller.HelloController.name=test-lixin
logging.config=classpath:logback-spring.xml
```

### (9). ProviderApplication
```
package help.lixin.refresh.bean;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class ProviderApplication {
	public static void main(String[] args) throws Exception {
		SpringApplication.run(ProviderApplication.class, args);
	}
}
```
### (10). 测试
```
// 1. 发起hello请求,这时读取的属性是配置文件里的,对象的地址是:help.lixin.refresh.bean.controller.HelloController@1518c72e
lixin-macbook:~ lixin$ curl http://localhost:8080/hello
Hello World!!! test-lixin -- help.lixin.refresh.bean.controller.HelloController@1518c72e

// 2. 发起refresh请求,刷新配置(help.lixin.refresh.bean.controller.RefreshController@37ed021c)
lixin-macbook:~ lixin$ curl http://localhost:8080/refresh
SUCCESS help.lixin.refresh.bean.controller.RefreshController@37ed021c

// 3. 发起hello请求,这时读取的配置文件变成热刷新的配置,对象的地址是:help.lixin.refresh.bean.controller.HelloController@2de634c5
lixin-macbook:~ lixin$ curl http://localhost:8080/hello
Hello World!!! lixin + f0b4918d-57f5-4adf-9457-aa47e888bf31 -- help.lixin.refresh.bean.controller.HelloController@2de634c5

// 这一步是求证,refresh("helloController")不会销毁RefreshController对象
// 4. 发起refresh请求,刷新配置(help.lixin.refresh.bean.controller.RefreshController@37ed021c)
lixin-macbook:~ lixin$ curl http://localhost:8080/refresh
SUCCESS help.lixin.refresh.bean.controller.RefreshController@37ed021c

```
### (11). 结论
> RefreshScope继承了:GenericScope,并且,实现了:ApplicationContextAware.   
> GenericScope内部持有所有Bean的实例(单例),以及相关的方法(destroy/get)   