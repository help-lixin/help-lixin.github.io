---
layout: post
title: 'Spring Cloud Stream集成Kafka(十)'
date: 2021-10-01
author: 李新
tags: SpringCloudStream Kafka
---

### (1). 概述
在这一小节,主要学习Spring Cloud Stream与Kafka的整合,后面会去剖析业务模型和部份源码.

### (2). 项目结构如下
> 生产者负责生产消息,消费者负责消费消息.  

```
lixin-macbook:Workspace lixin$ tree spring-cloud-stream-kafka-parent
spring-cloud-stream-kafka-parent
├── pom.xml
├── spring-cloud-stream-kafka-consumer                 # 消费者消费者
│   ├── pom.xml
│   ├── src
│   │   ├── main
│   │   │   ├── java
│   │   │   │   └── help
│   │   │   │       └── lixin
│   │   │   │           └── example
│   │   │   │               └── stream
│   │   │   │                   ├── ConsumerApplication.java
│   │   │   │                   └── service
│   │   │   │                       └── RecieveService.java
│   │   │   └── resources
│   │   │       └── application.properties
│   │   └── test
│   │       └── java
│   └── target
├── spring-cloud-stream-kafka-provider               # 消息生产者
│   ├── pom.xml
│   ├── src
│   │   ├── main
│   │   │   ├── java
│   │   │   │   └── help
│   │   │   │       └── lixin
│   │   │   │           └── example
│   │   │   │               └── stream
│   │   │   │                   ├── ProviderApplication.java
│   │   │   │                   ├── controller
│   │   │   │                   │   └── ProducerController.java
│   │   │   │                   └── service
│   │   │   │                       └── SendService.java
│   │   │   └── resources
│   │   │       └── application.properties
│   │   └── test
│   │       └── java
│   └── target
└── src
    ├── main
    │   ├── java
    │   └── resources
    └── test
        └── java
```

### (3). 引入依赖
```
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
		<groupId>org.springframework.cloud</groupId>
		<artifactId>spring-cloud-stream-binder-kafka</artifactId>
		<exclusions>
			<exclusion>
				<groupId>org.apache.kafka</groupId>
				<artifactId>kafka-clients</artifactId>
			</exclusion>
		</exclusions>
	</dependency>
	<dependency>
		<groupId>org.apache.kafka</groupId>
		<artifactId>kafka-clients</artifactId>
		<version>2.8.1</version>
	</dependency>
</dependencies>
```
### (4). SendService
```
package help.lixin.example.stream.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.annotation.EnableBinding;
import org.springframework.cloud.stream.messaging.Source;
import org.springframework.messaging.support.MessageBuilder;

@EnableBinding(Source.class)
public class SendService {

	@Autowired
	private Source source;

	public void sendMsg(String msg) {
		source.output().send(MessageBuilder.withPayload(msg).build());
	}
}
```
### (5). ProducerController
```
package help.lixin.example.stream.controller;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import help.lixin.example.stream.service.SendService;

@RestController
public class ProducerController {

	@Autowired
	private SendService sendService;

	@RequestMapping("/send/{msg}")
	public void send(@PathVariable("msg") String msg) {
		sendService.sendMsg(msg);
	}
}
```
### (6). application.properties
```
server.port=6060
spring.application.name=stream-provider

# kafka定义
spring.cloud.stream.kafka.binder.brokers=127.0.0.1:9092,127.0.0.1:9093,127.0.0.1:9094
spring.cloud.stream.kafka.binder.zk-nodes=127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183
spring.cloud.stream.kafka.binder.auto-create-topics=true
spring.cloud.stream.kafka.binder.required-acks=-1

# 定义channel(output)与topic的关系.
spring.cloud.stream.bindings.output.destination=hello-world
spring.cloud.stream.bindings.output.content-type=text/plain
```
### (7). ProviderApplication
```
package help.lixin.example.stream;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class ProviderApplication {
	public static void main(String[] args) {
		SpringApplication.run(ProviderApplication.class, args);
	}
}
```
### (7). RecieveService
```
package help.lixin.example.stream.service;

import org.springframework.cloud.stream.annotation.EnableBinding;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.cloud.stream.messaging.Sink;

@EnableBinding(Sink.class)
public class RecieveService {

	@StreamListener(Sink.INPUT)
	public void recieve(String payload) {
		System.out.println(payload);
	}
}
```
### (8). application.properties
```
server.port=6061
spring.application.name=stream-consumer

spring.cloud.stream.kafka.binder.brokers=127.0.0.1:9092,127.0.0.1:9093,127.0.0.1:9094
spring.cloud.stream.kafka.binder.auto-create-topics=true

spring.cloud.stream.bindings.input.destination=hello-world
#spring.cloud.stream.kafka.bindings.input.group=${spring.application.name}
```
### (9). ConsumerApplication
```
package help.lixin.example.stream;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class ConsumerApplication {
	public static void main(String[] args) {
		SpringApplication.run(ConsumerApplication.class, args);
	}
}
```
### (10). 查看kafka所有topic
```
lixin-macbook:bin lixin$ ./kafka-topics.sh  --zookeeper 127.0.0.1:2181  --list
__consumer_offsets
hello-world
```
### (11). 测试运行
```
lixin-macbook:~ lixin$ curl http://localhost:6060/send/hello
lixin-macbook:~ lixin$ curl http://localhost:6060/send/world
```
### (12). 总结
通过Spring Cloud Stream之后,我们可以做到在代码级别无缝的切换MQ厂商.  