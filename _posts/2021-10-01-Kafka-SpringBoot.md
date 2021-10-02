---
layout: post
title: 'Spring Boot与Kafka集成(六)' 
date: 2021-10-01
author: 李新
tags:  Kafka
---

### (1). 概述
这一小节,主要把Kafka与SpringBoot进行集成,并能发送和接受消息.   
### (2). 项目结构
```
lixin-macbook:kafka-example lixin$ tree 
.
├── pom.xml
├── src
│   ├── main
│   │   ├── java
│   │   │   └── help
│   │   │       └── lixin
│   │   │           └── kafka
│   │   │               └── example
│   │   │                   ├── KafkaApplication.java
│   │   │                   ├── consumer
│   │   │                   │   └── MessageRevice.java
│   │   │                   └── controller
│   │   │                       └── MessageController.java
│   │   └── resources
│   │       └── application.yml
└── test-classes
```
### (3). 引入依赖(pom.xml)
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>help.lixin.kafka</groupId>
    <artifactId>kafka-example</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
        <spring-cloud.version>Finchley.RC2</spring-cloud.version>
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
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka</artifactId>
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
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.4</version>
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
### (4). 定义配置(application.yml)文件
```
server:
  port: 8088

spring:
  kafka:
    bootstrap-servers: 127.0.0.1:9092,127.0.0.1:9093,127.0.0.1:9094
    producer:
      retries: 3
      batch-size: 16384
      buffer-memory: 33554432
      acsk: -1
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
    consumer:
      group-id: test-service-1
      enable-auto-commit: false
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      max-poll-records: 500
    listener:
#      ack-mode: RECORD   #
#      ack-mode: RECORD
#      ack-mode: BATCH
#      ack-mode: TIME
#      ack-mode: COUNT
#      ack-mode: COUNT_TIME
#       手动调用Acknowledgment.acknowledge()后立即提交
      ack-mode: MANUAL_IMMEDIATE
```
### (5). MessageController
```
package help.lixin.kafka.example.controller;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class MessageController {

    private String topic = "hello";

    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;

    @GetMapping("/send")
    public String send() {
        kafkaTemplate.send(topic, "Hello World!!!!");
        return "SUCCESS";
    }
}
```
### (6). MessageRevice
```
package help.lixin.kafka.example.consumer;

import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.support.Acknowledgment;
import org.springframework.stereotype.Component;

@Component
public class MessageRevice {
    
	// concurrency : 代表为:test-group配置2个Consumer去消费,建议:与Partition数量相同.
	// @KafkaListener(topics = {"${listener.topic}"}, groupId = "test-group", concurrency = "${listener.concurrency}")
    @KafkaListener(topics = {"hello"}, groupId = "test-group", concurrency = "2")
    public void listen(ConsumerRecord<String, String> record, Acknowledgment ack) {
        System.out.println("Revice Message: " + record.value());
        // 手动提交offset
        ack.acknowledge();
    }
}
```
### (7). KafkaApplication
```
package help.lixin.kafka.example;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class KafkaApplication {
    public static void main(String[] args) {
        SpringApplication.run(KafkaApplication.class, args);
    }
}
```
### (8). 总结
Kafka与Spring的整合之后,相当的简单,不过,过于简单的东西就会失去控制性,所以,后面的章节会对Kafka和SpringBoot的整合源码进行剖解.  