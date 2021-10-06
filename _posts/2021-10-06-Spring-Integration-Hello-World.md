---
layout: post
title: 'Spring Integration简单入门案例(二)' 
date: 2021-10-06
author: 李新
tags:  SpringIntegration
---

### (1). 概述
在这一篇,主要是对Spring Integration有一个入门的案例,这个案例,通过Greeting发送消息到:MessageChannel,并调用:HelloMessageProvider返回结果.  

### (2). 引入依赖
```
<dependency>
	<groupId>org.springframework.integration</groupId>
	<artifactId>spring-integration-core</artifactId>
	<version>4.3.5.RELEASE</version>
</dependency>						
```
### (3). 配置Spring Integeration(hell-world.xml)
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:int="http://www.springframework.org/schema/integration"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/integration
        http://www.springframework.org/schema/integration/spring-integration.xsd">
    <int:gateway default-request-channel="inChannel" service-interface="help.lixin.integration.example.Application$Greeting"/>
    <int:channel id="inChannel"/>
    <int:service-activator input-channel="inChannel" method="sayHello">
        <bean class="help.lixin.integration.example.Application$HelloMessageProvider"/>
    </int:service-activator>
</beans>
```
### (4). Application
```
package help.lixin.integration.example;

import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.support.ClassPathXmlApplicationContext;
import org.springframework.integration.annotation.Gateway;
import org.springframework.integration.annotation.IntegrationComponentScan;
import org.springframework.integration.annotation.MessagingGateway;
import org.springframework.integration.annotation.ServiceActivator;
import org.springframework.integration.channel.DirectChannel;
import org.springframework.integration.config.EnableIntegration;
import org.springframework.messaging.MessageChannel;
import org.springframework.stereotype.Component;

@Configuration
@EnableIntegration
@IntegrationComponentScan
public class Application {
    
	@MessagingGateway
	interface Greeting {
		@Gateway(requestChannel = "inChannel")
		String greet(String name);
	}
	
	// ************************************************************
	// 定义:MessageChannel
	// ************************************************************
	@Bean
	MessageChannel inChannel() {
		return new DirectChannel();
	}

	// ServiceActivator
	@Component
	static class HelloMessageProvider {
		@ServiceActivator(inputChannel = "inChannel")
		public String sayHello(String name) {
			return "Hi, " + name;
		}
	}

	public static void main(String[] args) {
		// 通过注解初始化
		// ApplicationContext context = new AnnotationConfigApplicationContext(Application.class);
		//  通过XML始化
		ApplicationContext context = new ClassPathXmlApplicationContext("classpath:hello-world.xml");
		Greeting greeting = context.getBean(Greeting.class);
		System.out.println(greeting.greet("Spring Integration!"));
	}
}

```
### (5). 总结
我们,先运行这个案例,后面,会根据这个案例对Spring Integration源码进行剖析.  