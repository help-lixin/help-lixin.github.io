---
layout: post
title: 'Eventuate事件发布' 
date: 2023-03-24
author: 李新
tags:  Eventuate
---

### (1). 背景
在某些业务场景下,我们会用到消息中间件,那么,在用消息中间件在发送消息时,如何保证100%的不丢消息呢,那就需要事务消息了.   
最近在研究Eventuate,最初没打算用这套框架,但是,想了想吧!还是少自研一些,尽量向别人的开源框架靠拢,如果不满足自己做扩展,这样,有好处,也有坏处,坏处就是我需要花大量的时间看完源码. 
### (2). 项目结构
```
lixin-macbook:event-example lixin$ tree
.
├── event-api-message                    # 生产者\消费者共用的业务模型和事件
│   ├── pom.xml
│   └── src
│       └── main
│           ├── java
│           │   └── help
│           │       └── lixin
│           │           └── domain
│           │               ├── Account.java
│           │               └── AccountDebited.java
│           └── resources
├── event-consumer                         # 消费者模块
│   ├── pom.xml
│   └── src
│       └── main
│           ├── java
│           │   └── help
│           │       └── lixin
│           │           └── consumer
│           │               ├── EventConsumerApp.java
│           │               ├── config
│           │               │   └── EventConsumerConfig.java
│           │               └── service
│           │                   ├── AccountEventConsumer.java
│           │                   ├── AggregateSupplier.java
│           │                   └── IdSupplier.java
│           └── resources
│               └── application.yaml
├── event-producer                          # 生产者模块
│   ├── pom.xml
│   └── src
│       └── main
│           ├── java
│           │   └── help
│           │       └── lixin
│           │           └── producer
│           │               ├── EventProducerApp.java
│           │               ├── config
│           │               │   └── ProducerConfig.java
│           │               └── controller
│           │                   └── PublisherController.java
│           └── resources
│               └── application.yaml
├── pom.xml
└── sql                                     # sql脚本
    └── eventuate.sql
```
### (3). eventuate.sql
```

SET NAMES utf8mb4;
SET FOREIGN_KEY_CHECKS = 0;

-- ----------------------------
-- Table structure for cdc_monitoring
-- ----------------------------
DROP TABLE IF EXISTS `cdc_monitoring`;
CREATE TABLE `cdc_monitoring` (
  `reader_id` varchar(255) NOT NULL,
  `last_time` bigint DEFAULT NULL,
  PRIMARY KEY (`reader_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- ----------------------------
-- Table structure for entities
-- ----------------------------
DROP TABLE IF EXISTS `entities`;
CREATE TABLE `entities` (
  `entity_type` varchar(255) NOT NULL,
  `entity_id` varchar(255) NOT NULL,
  `entity_version` longtext NOT NULL,
  PRIMARY KEY (`entity_type`,`entity_id`),
  KEY `entities_idx` (`entity_type`,`entity_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- ----------------------------
-- Table structure for events
-- ----------------------------
DROP TABLE IF EXISTS `events`;
CREATE TABLE `events` (
  `event_id` varchar(255) NOT NULL,
  `event_type` longtext,
  `event_data` longtext CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL,
  `entity_type` varchar(255) NOT NULL,
  `entity_id` varchar(255) NOT NULL,
  `triggering_event` longtext,
  `metadata` longtext,
  `published` tinyint DEFAULT '0',
  PRIMARY KEY (`event_id`),
  KEY `events_idx` (`entity_type`,`entity_id`,`event_id`),
  KEY `events_published_idx` (`published`,`event_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- ----------------------------
-- Table structure for message
-- ----------------------------
DROP TABLE IF EXISTS `message`;
CREATE TABLE `message` (
  `id` varchar(255) NOT NULL,
  `destination` longtext NOT NULL,
  `headers` longtext CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL,
  `payload` longtext CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL,
  `published` smallint DEFAULT '0',
  `message_partition` smallint DEFAULT NULL,
  `creation_time` bigint DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `message_published_idx` (`published`,`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- ----------------------------
-- Table structure for offset_store
-- ----------------------------
DROP TABLE IF EXISTS `offset_store`;
CREATE TABLE `offset_store` (
  `client_name` varchar(255) NOT NULL,
  `serialized_offset` longtext,
  PRIMARY KEY (`client_name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- ----------------------------
-- Table structure for received_messages
-- ----------------------------
DROP TABLE IF EXISTS `received_messages`;
CREATE TABLE `received_messages` (
  `consumer_id` varchar(255) NOT NULL,
  `message_id` varchar(255) NOT NULL,
  `creation_time` bigint DEFAULT NULL,
  `published` smallint DEFAULT '0',
  PRIMARY KEY (`consumer_id`,`message_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- ----------------------------
-- Table structure for saga_instance
-- ----------------------------
DROP TABLE IF EXISTS `saga_instance`;
CREATE TABLE `saga_instance` (
  `saga_type` varchar(255) NOT NULL,
  `saga_id` varchar(100) NOT NULL,
  `state_name` varchar(100) NOT NULL,
  `last_request_id` varchar(100) DEFAULT NULL,
  `end_state` int DEFAULT NULL,
  `compensating` int DEFAULT NULL,
  `failed` int DEFAULT NULL,
  `saga_data_type` varchar(1000) NOT NULL,
  `saga_data_json` varchar(1000) NOT NULL,
  PRIMARY KEY (`saga_type`,`saga_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- ----------------------------
-- Table structure for saga_instance_participants
-- ----------------------------
DROP TABLE IF EXISTS `saga_instance_participants`;
CREATE TABLE `saga_instance_participants` (
  `saga_type` varchar(255) NOT NULL,
  `saga_id` varchar(100) NOT NULL,
  `destination` varchar(100) NOT NULL,
  `resource` varchar(100) NOT NULL,
  PRIMARY KEY (`saga_type`,`saga_id`,`destination`,`resource`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- ----------------------------
-- Table structure for saga_lock_table
-- ----------------------------
DROP TABLE IF EXISTS `saga_lock_table`;
CREATE TABLE `saga_lock_table` (
  `target` varchar(100) NOT NULL,
  `saga_type` varchar(255) NOT NULL,
  `saga_Id` varchar(100) NOT NULL,
  PRIMARY KEY (`target`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- ----------------------------
-- Table structure for saga_stash_table
-- ----------------------------
DROP TABLE IF EXISTS `saga_stash_table`;
CREATE TABLE `saga_stash_table` (
  `message_id` varchar(100) NOT NULL,
  `target` varchar(100) NOT NULL,
  `saga_type` varchar(255) NOT NULL,
  `saga_id` varchar(100) NOT NULL,
  `message_headers` varchar(1000) NOT NULL,
  `message_payload` varchar(1000) NOT NULL,
  PRIMARY KEY (`message_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- ----------------------------
-- Table structure for snapshots
-- ----------------------------
DROP TABLE IF EXISTS `snapshots`;
CREATE TABLE `snapshots` (
  `entity_type` varchar(255) NOT NULL,
  `entity_id` varchar(255) NOT NULL,
  `entity_version` varchar(255) NOT NULL,
  `snapshot_type` longtext NOT NULL,
  `snapshot_json` longtext NOT NULL,
  `triggering_events` longtext,
  PRIMARY KEY (`entity_type`,`entity_id`,`entity_version`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

SET FOREIGN_KEY_CHECKS = 1;
```
### (4). 定义业务模型
```
package help.lixin.domain;

public class Account {

}
```
### (5). DomainEvent
```
package help.lixin.domain;

import io.eventuate.tram.events.common.DomainEvent;

public class AccountDebited implements DomainEvent {
    private long amount;

    public AccountDebited() {
    }

    public AccountDebited(long amount) {

        this.amount = amount;
    }

    public long getAmount() {
        return amount;
    }

    public void setAmount(long amount) {
        this.amount = amount;
    }
}
```
### (6). 为业务模型工程添加依赖
```
<dependency>
	<groupId>io.eventuate.tram.core</groupId>
	<artifactId>eventuate-tram-events</artifactId>
	<version>${eventuate-tram.version}</version>
	<scope>provided</scope>
</dependency>
```
### (7). 定义生产者配置
```
package help.lixin.producer.config;

import io.eventuate.tram.spring.events.publisher.TramEventsPublisherConfiguration;
import io.eventuate.tram.spring.messaging.producer.jdbc.TramMessageProducerJdbcConfiguration;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Import;

@Configuration
@Import({TramEventsPublisherConfiguration.class,
        //
        TramMessageProducerJdbcConfiguration.class})
public class ProducerConfig {

}
```
### (8). 定义生产者发布事件
```
package help.lixin.producer.controller;


import help.lixin.domain.Account;
import help.lixin.domain.AccountDebited;
import io.eventuate.tram.events.common.DomainEvent;
import io.eventuate.tram.events.publisher.DomainEventPublisher;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.Collections;

@RestController
public class PublisherController {

    @Autowired
    private DomainEventPublisher domainEventPublisher;


    @GetMapping("/publish")
    public String publish() {
        String aggregateType = Account.class.getName();
        Long aggregateId = System.currentTimeMillis();
        Long uniqueId = aggregateId;
        DomainEvent domainEvent = new AccountDebited(uniqueId);
        // 发布事件.
        domainEventPublisher.publish(
                //
                aggregateType,
                //
                uniqueId,
                //
                Collections.singletonList(domainEvent));
        return "SUCCESS";
    }
}
```
### (9). 生产者配置文件
> 注意:生产者是强依赖于db来着的

```
server:
  port: 9091

spring:
  datasource:
    url: jdbc:mysql://127.0.0.1:3306/eventuate
    username: root
    password: 123456
    driver-class-name: com.mysql.jdbc.Driver
```
### (10). 生产者依赖配置
```
<dependency>
	<groupId>help.lixin</groupId>
	<artifactId>event-api-message</artifactId>
	<version>${project.version}</version>
</dependency>


<dependency>
	<groupId>mysql</groupId>
	<artifactId>mysql-connector-java</artifactId>
</dependency>

<dependency>
	<groupId>io.eventuate.tram.core</groupId>
	<artifactId>eventuate-tram-events</artifactId>
	<version>${eventuate-tram.version}</version>
</dependency>

<dependency>
	<groupId>io.eventuate.tram.core</groupId>
	<artifactId>eventuate-tram-spring-producer-jdbc</artifactId>
	<version>${eventuate-tram.version}</version>
</dependency>

<dependency>
	<groupId>io.eventuate.tram.core</groupId>
	<artifactId>eventuate-tram-spring-events-publisher</artifactId>
	<version>${eventuate-tram.version}</version>
</dependency>

<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
```
### (11). 定义消费者配置类
```
package help.lixin.consumer.config;

import help.lixin.consumer.service.AccountEventConsumer;
import help.lixin.domain.Account;
import io.eventuate.tram.consumer.common.DuplicateMessageDetector;
import io.eventuate.tram.consumer.common.NoopDuplicateMessageDetector;
import io.eventuate.tram.events.subscriber.DomainEventDispatcher;
import io.eventuate.tram.events.subscriber.DomainEventDispatcherFactory;
import io.eventuate.tram.spring.consumer.kafka.EventuateTramKafkaMessageConsumerConfiguration;
import io.eventuate.tram.spring.events.subscriber.TramEventSubscriberConfiguration;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Import;

@Configuration
@Import({TramEventSubscriberConfiguration.class,
        EventuateTramKafkaMessageConsumerConfiguration.class})
public class EventConsumerConfig {

    @Bean
    public DuplicateMessageDetector noopDuplicateMessageDetector() {
        return new NoopDuplicateMessageDetector();
    }

    @Bean
    public AccountEventConsumer accountEventConsumer() {
        return new AccountEventConsumer(Account.class.getName());
    }

    @Bean
    public DomainEventDispatcher domainEventDispatcher(
            // 定义事件分发工厂,实际内部是持有一个MessageConsumer
            DomainEventDispatcherFactory domainEventDispatcherFactory,
            // 引用上面的:AccountEventConsumer
            AccountEventConsumer target) {
        return domainEventDispatcherFactory.make("eventDispatcherId", target.domainEventHandlers());
    } // end
}
```
### (12). 定义消费者类
```
package help.lixin.consumer.service;

import help.lixin.domain.AccountDebited;
import io.eventuate.tram.events.subscriber.DomainEventEnvelope;
import io.eventuate.tram.events.subscriber.DomainEventHandlers;
import io.eventuate.tram.events.subscriber.DomainEventHandlersBuilder;

public class AccountEventConsumer {
    private final String aggregateType;

    public AccountEventConsumer(String aggregateType) {
        this.aggregateType = aggregateType;
    }

    public DomainEventHandlers domainEventHandlers() {
        return DomainEventHandlersBuilder
                // 聚合根对象:Account
                .forAggregateType(aggregateType)
                // 事件处理
                .onEvent(AccountDebited.class, this::handleAccountDebited)
                //
                .build();
    }

    public void handleAccountDebited(DomainEventEnvelope<AccountDebited> event) {
        System.out.println(event);
    }
}
```
### (13). 定义消费者配置类
```
server:
  port: 9092

eventuatelocal:
  kafka:
    bootstrap:
      servers: 127.0.0.1:9092
```
### (14). 定义消费者依赖
```
<dependency>
	<groupId>help.lixin</groupId>
	<artifactId>event-api-message</artifactId>
	<version>${project.version}</version>
</dependency>

<dependency>
	<groupId>io.eventuate.tram.core</groupId>
	<artifactId>eventuate-tram-events</artifactId>
	<version>${eventuate-tram.version}</version>
</dependency>

<dependency>
	<groupId>io.eventuate.tram.core</groupId>
	<artifactId>eventuate-tram-spring-events-subscriber</artifactId>
	<version>${eventuate-tram.version}</version>
</dependency>

<dependency>
	<groupId>io.eventuate.tram.core</groupId>
	<artifactId>eventuate-tram-spring-consumer-kafka</artifactId>
	<version>${eventuate-tram.version}</version>
</dependency>

<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
```
### (15). 配置启动:eventuate-cdc-service
```
#!/bin/bash

export SPRING_DATASOURCE_DRIVER_CLASS_NAME=com.mysql.jdbc.Driver
export SPRING_DATASOURCE_URL=jdbc:mysql://localhost:3306/eventuate

export SPRING_DATASOURCE_USERNAME=root
export SPRING_DATASOURCE_PASSWORD=123456
export EVENTUATELOCAL_CDC_DB_USER_NAME=root
export EVENTUATELOCAL_CDC_DB_PASSWORD=123456

export JAVA_OPTS=-Xmx64m

export EVENTUATELOCAL_CDC_OFFSET_STORE_KEY=MySqlBinlog
export EVENTUATELOCAL_CDC_READ_OLD_DEBEZIUM_DB_OFFSET_STORAGE_TOPIC=false
# EVENTUATE_CDC_OUTBOX_PARTITIONING_OUTBOX_TABLES=

export EVENTUATELOCAL_CDC_READER_NAME=MySqlReader
export JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk-18.0.1.1.jdk/Contents/Home
export EVENTUATELOCAL_CDC_MYSQL_BINLOG_CLIENT_UNIQUE_ID=1234567890

export EVENTUATELOCAL_KAFKA_BOOTSTRAP_SERVERS=localhost:9092
export EVENTUATELOCAL_ZOOKEEPER_CONNECTION_STRING=localhost:2181
export EVENTUATE_OUTBOX_ID=1


# 需要自己去拉取源码下来,然后,编译.
# https://github.com/eventuate-foundation/eventuate-cdc
java -jar ./eventuate-cdc-service-0.15.0-SNAPSHOT.jar -Xmx256m
```
### (16). 启动项目
```
1. 启动mysql(注意:mysql要配置开启binlong)
2. 启动zookeeper
3. 启动kafka
4. 启动eventuate-cdc-service
5. 启动producer
6. 启动consumer
```
### (17). 测试
```
lixin-macbook:~ lixin$ curl http://localhost:9091/publish
SUCCESS
```
### (18). 验证
验证消费者端是否有消息消费成功. 