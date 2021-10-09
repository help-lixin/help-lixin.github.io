---
layout: post
title: 'Spring Integration简单入门案例(二)' 
date: 2021-10-06
author: 李新
tags:  SpringIntegration
---

### (1). 概述
在这一篇,主要是对Spring Integration有一个入门的案例,这个案例,来自于[官网](https://github.com/help-lixin/spring-integration-samples/tree/main/basic/helloworld)   

### (2). 项目结构如如下
```
lixin@lixin basic % tree helloworld 
helloworld
├── pom.xml
└── src
    └── main
        ├── java
        │   └── org
        │       └── springframework
        │           └── integration
        │               └── samples
        │                   └── helloworld
        │                       ├── HelloService.java
        │                       └── HelloWorldApp.java
        └── resources
            ├── META-INF
            │   └── spring
            │       └── integration
            │           └── helloWorldDemo.xml
            └── log4j.xml
```

### (3). 引入依赖
```
<dependency>
	<groupId>org.springframework.integration</groupId>
	<artifactId>spring-integration-core</artifactId>
	<version>4.3.5.RELEASE</version>
</dependency>						
```
### (4). 配置Spring Integeration(helloWorldDemo.xml)
```
<?xml version="1.0" encoding="UTF-8"?>
<beans:beans xmlns="http://www.springframework.org/schema/integration"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:beans="http://www.springframework.org/schema/beans"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
			https://www.springframework.org/schema/beans/spring-beans.xsd
			http://www.springframework.org/schema/integration
			https://www.springframework.org/schema/integration/spring-integration.xsd">
	<channel id="inputChannel"/>
	<channel id="outputChannel">
		<queue capacity="10"/>
	</channel>
	<service-activator input-channel="inputChannel"
	                   output-channel="outputChannel"
	                   ref="helloService"
	                   method="sayHello"/>
	<beans:bean id="helloService" class="org.springframework.integration.samples.helloworld.HelloService"/>
</beans:beans>

```
### (5). HelloService
```
package org.springframework.integration.samples.helloworld;

public class HelloService {

	public String sayHello(String name) {
		return "Hello " + name;
	}

}
```
### (6). HelloWorldApp
```
package org.springframework.integration.samples.helloworld;

import org.apache.log4j.Logger;
import org.springframework.context.support.AbstractApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.PollableChannel;
import org.springframework.messaging.support.GenericMessage;

public class HelloWorldApp {
	private static Logger logger = Logger.getLogger(HelloWorldApp.class);

	public static void main(String[] args) {
		AbstractApplicationContext context = new ClassPathXmlApplicationContext("/META-INF/spring/integration/helloWorldDemo.xml", HelloWorldApp.class);
		// 从上下文中,获得:MessageChannel
		MessageChannel inputChannel = context.getBean("inputChannel", MessageChannel.class);
		// 从上下文中,获得:PollableChannel
		PollableChannel outputChannel = context.getBean("outputChannel", PollableChannel.class);
		// 发送一条消息
		inputChannel.send(new GenericMessage<String>("World"));
		// 接收消息
		logger.info("==> HelloWorldDemo: " + outputChannel.receive(0).getPayload());
	}

}
```
### (7). 运行结果
```
11:18:26.582 INFO  [main][org.springframework.integration.samples.helloworld.HelloWorldApp] ==> HelloWorldDemo: Hello World
```
### (8). 总结
我们,先运行这个案例,后面,会根据这个案例对Spring Integration源码进行剖析.  