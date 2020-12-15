---
layout: post
title: 'ElasticSearch 源码Inject(五)'
date: 2020-12-14
author: 李新
tags: ElasticSearch源码
---

### (1). ElasticSearch依赖注入
> 一直以来,以为:ES的依赖注入使用了:Google Guice,深入跟踪了半天,发现并不我以为的那样(对Guice进行了包装),而是,<font color='red'>参照Guice自己写了一套:DI.</font>感觉在自己造轮子,不过,也对哈!人家要的东西小巧玲珑.  
> 下面是我从ES中抽出来的依赖注入.并做的测试,等以后有时间再把依赖注入的源码分析. 

### (2). 项目结构图如下
!["ES依赖注入项目结构图"](/assets/elasticsearch/imgs/es-inject-project.jpg)

["ES依赖注入项目下载"](/assets/elasticsearch/inject-core.zip)

### (3). 定义接口(IHelloService)
```
package help.lixin.common.service;

public interface IHelloService {
	void sayHello();
}
```
### (4). 定义接口实现(HelloService)
```
package help.lixin.common.service.impl;

import help.lixin.common.inject.Inject;
import help.lixin.common.inject.name.Named;
import help.lixin.common.service.IHelloService;
import help.lixin.common.util.CommonLog;

public class HelloService implements IHelloService {

	private CommonLog commonLog;
	private int port;

    // *****************************************************************
    // @Inject : 依赖注入对象
    // @Named  : 根据名称注入
    // *****************************************************************
	@Inject
	public HelloService(CommonLog commonLog, @Named("port") int port) {
		this.commonLog = commonLog;
		this.port = port;
		this.commonLog.println("=====================HelloService init========================");
	}

	@Override
	public void sayHello() {
		commonLog.println("*********************start bind port:" + port + "*****************");
		commonLog.println("*********************say Hello*****************");
	}
}
```
### (5). 定义公共日志输出(CommonLog)
```
package help.lixin.common.util;

import java.io.PrintStream;

public class CommonLog {
	private static final PrintStream OUT = System.out;

	public void println(String line) {
		OUT.append(line).append("\n");
	}
}
```
### (6). 定义Module(CustomerModule)
> 类似于Spring中书写XML(Configuration)

```
package help.lixin.common.module;

import help.lixin.common.inject.AbstractModule;
import help.lixin.common.inject.Singleton;
import help.lixin.common.inject.name.Names;
import help.lixin.common.service.IHelloService;
import help.lixin.common.service.impl.HelloService;
import help.lixin.common.util.CommonLog;

public class CustomerModule extends AbstractModule {

	@Override
	protected void configure() {
        // 定义接口与实现类
		bind(IHelloService.class).to(HelloService.class).in(Singleton.class);
        // 定义类对应的实例
		bind(CommonLog.class).toInstance(new CommonLog());
        // 定义常量
		bindConstant().annotatedWith(Names.named("port")).to(8080);
	}
}
```
### (7). 测试HelloService(InjectTest)
> 目标:测试HelloService,看是否会自动注入:CommonLog commonLog和int port

```
package help.lixin.common.inject;

import org.junit.Test;

import help.lixin.common.module.CustomerModule;
import help.lixin.common.service.IHelloService;

public class InjectTest {
	
	@Test
	public void testInject() {
		
		Module module = new CustomerModule();
		
		ModulesBuilder builder = new ModulesBuilder();
		builder.add(module);
		
        // 创建:Injector,并设置对象与对象之间的依赖
		Injector inject = builder.createInjector();
		
		// 获取实例
		IHelloService helloService = inject.getInstance(IHelloService.class);
		helloService.sayHello();
	}
}
```
### (8). 查看控制台输出 
```
=====================HelloService init========================
*********************start bind port:8080*****************
*********************say Hello*****************
```

### (9). 项目pom.xml
> 并不需要依赖:guice.   

```
<project xmlns="http://maven.apache.org/POM/4.0.0"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>help.lixin.inject</groupId>
	<artifactId>inject-core</artifactId>
	<version>1.0.0-SNAPSHOT</version>
	<packaging>jar</packaging>

	<name>inject-core</name>
	<url>http://maven.apache.org</url>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<maven.compiler.source>1.8</maven.compiler.source>
		<maven.compiler.target>1.8</maven.compiler.target>
		<java.version>1.8</java.version>
	</properties>

	<dependencies>
		<dependency>
			<groupId>org.apache.logging.log4j</groupId>
			<artifactId>log4j-api</artifactId>
			<version>2.11.1</version>
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