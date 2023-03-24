---
layout: post
title: 'Eventuate环境搭建' 
date: 2023-03-24
author: 李新
tags:  Eventuate
---

### (1). 概述

最近在研究Eventuate,其实,最初没打算用这套框架,但是,想了想吧!还是少自研一些,尽量向别人的框架靠拢,如果不满足自己做扩展,这样,有好处,也有坏处,坏处就是我需要花大量的时间看完源码,如何入手呢?官网有一个Demo,经历漫长的拉取镜像和调试,终于是成功了的,特意摘抄这部份内容,方便后面从Docker容器迁本机做测试.    

### (2). cdc-service环境变量配置
```
# 数据库配置
SPRING_DATASOURCE_DRIVER_CLASS_NAME=com.mysql.jdbc.Driver
SPRING_DATASOURCE_URL=jdbc:mysql://localhost:3306/eventuate

SPRING_DATASOURCE_USERNAME=mysqluser
SPRING_DATASOURCE_PASSWORD=mysqlpw
EVENTUATELOCAL_CDC_DB_USER_NAME=root
EVENTUATELOCAL_CDC_DB_PASSWORD=rootpassword

JAVA_OPTS=-Xmx64m

# 读取数据的名称
EVENTUATELOCAL_CDC_OFFSET_STORE_KEY=MySqlBinlog
EVENTUATELOCAL_CDC_READ_OLD_DEBEZIUM_DB_OFFSET_STORAGE_TOPIC=false
EVENTUATE_CDC_OUTBOX_PARTITIONING_OUTBOX_TABLES=

EVENTUATELOCAL_CDC_READER_NAME=MySqlReader
JAVA_HOME=/usr/lib/jvm/java-17-amazon-corretto
EVENTUATELOCAL_CDC_MYSQL_BINLOG_CLIENT_UNIQUE_ID=1234567890

# kafka配置
EVENTUATELOCAL_KAFKA_BOOTSTRAP_SERVERS=localhost:9092
# zk配置
EVENTUATELOCAL_ZOOKEEPER_CONNECTION_STRING=localhost:2181
EVENTUATE_OUTBOX_ID=1
```
### (3). order-service环境变量配置
```

# 依赖zipkin
SPRING_ZIPKIN_BASE_URL=http://localhost:9411/
SPRING_SLEUTH_SAMPLER_PROBABILITY=1

# 依赖db
SPRING_DATASOURCE_DRIVER_CLASS_NAME=com.mysql.jdbc.Driver
SPRING_DATASOURCE_URL=jdbc:mysql://localhost:3306/eventuate
SPRING_DATASOURCE_USERNAME=mysqluser
SPRING_DATASOURCE_PASSWORD=mysqlpw

EVENTUATE_TRAM_OUTBOX_PARTITIONING_OUTBOX_TABLES=

# 依赖kafka和zk
EVENTUATELOCAL_KAFKA_BOOTSTRAP_SERVERS=localhost:9092
EVENTUATELOCAL_ZOOKEEPER_CONNECTION_STRING=localhost:2181
```
### (4). customer-servie环境变量配置
```
SPRING_DATASOURCE_DRIVER_CLASS_NAME=com.mysql.jdbc.Driver
SPRING_DATASOURCE_URL=jdbc:mysql://localhost:3306/eventuate
SPRING_DATASOURCE_USERNAME=mysqluser
SPRING_DATASOURCE_PASSWORD=mysqlpw

EVENTUATE_TRAM_OUTBOX_PARTITIONING_OUTBOX_TABLES=

SPRING_SLEUTH_ENABLED=true
SPRING_ZIPKIN_BASE_URL=http://localhost:9411/
SPRING_SLEUTH_SAMPLER_PROBABILITY=1

EVENTUATELOCAL_KAFKA_BOOTSTRAP_SERVERS=localhost:9092
EVENTUATELOCAL_ZOOKEEPER_CONNECTION_STRING=localhost:2181
```
### (5). api-gateway-service环境变量配置
```
APIGATEWAY_TIMEOUT_MILLIS=1000000

CUSTOMER_DESTINATIONS_CUSTOMERSERVICEURL=http://customer-service:8080
JAVA_OPTS=-Ddebug

SPRING_SLEUTH_ENABLED=true
SPRING_ZIPKIN_BASE_URL=http://localhost:9411/
SPRING_SLEUTH_SAMPLER_PROBABILITY=1

ORDER_DESTINATIONS_ORDERSERVICEURL=http://order-service:8080
```
### (6). sql脚本
```
-- MySQL dump 10.13  Distrib 8.0.27, for Linux (x86_64)
--
-- Host: localhost    Database: eventuate
-- ------------------------------------------------------
-- Server version 8.0.27

/*!40101 SET @OLD_CHARACTER_SET_CLIENT=@@CHARACTER_SET_CLIENT */;
/*!40101 SET @OLD_CHARACTER_SET_RESULTS=@@CHARACTER_SET_RESULTS */;
/*!40101 SET @OLD_COLLATION_CONNECTION=@@COLLATION_CONNECTION */;
/*!50503 SET NAMES utf8mb4 */;
/*!40103 SET @OLD_TIME_ZONE=@@TIME_ZONE */;
/*!40103 SET TIME_ZONE='+00:00' */;
/*!40014 SET @OLD_UNIQUE_CHECKS=@@UNIQUE_CHECKS, UNIQUE_CHECKS=0 */;
/*!40014 SET @OLD_FOREIGN_KEY_CHECKS=@@FOREIGN_KEY_CHECKS, FOREIGN_KEY_CHECKS=0 */;
/*!40101 SET @OLD_SQL_MODE=@@SQL_MODE, SQL_MODE='NO_AUTO_VALUE_ON_ZERO' */;
/*!40111 SET @OLD_SQL_NOTES=@@SQL_NOTES, SQL_NOTES=0 */;

--
-- Table structure for table `cdc_monitoring`
--

DROP TABLE IF EXISTS `cdc_monitoring`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!50503 SET character_set_client = utf8mb4 */;
CREATE TABLE `cdc_monitoring` (
  `reader_id` varchar(255) NOT NULL,
  `last_time` bigint DEFAULT NULL,
  PRIMARY KEY (`reader_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `cdc_monitoring`
--

LOCK TABLES `cdc_monitoring` WRITE;
/*!40000 ALTER TABLE `cdc_monitoring` DISABLE KEYS */;
INSERT INTO `cdc_monitoring` VALUES ('MySqlReader',1679552433042);
/*!40000 ALTER TABLE `cdc_monitoring` ENABLE KEYS */;
UNLOCK TABLES;

--
-- Table structure for table `customer`
--

DROP TABLE IF EXISTS `customer`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!50503 SET character_set_client = utf8mb4 */;
CREATE TABLE `customer` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `amount` decimal(19,2) DEFAULT NULL,
  `name` varchar(255) DEFAULT NULL,
  `version` bigint DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `customer`
--

LOCK TABLES `customer` WRITE;
/*!40000 ALTER TABLE `customer` DISABLE KEYS */;
INSERT INTO `customer` VALUES (1,15.00,'John',0),(2,15.00,'John',1),(3,5.00,'Jane Doe',1);
/*!40000 ALTER TABLE `customer` ENABLE KEYS */;
UNLOCK TABLES;

--
-- Table structure for table `customer_credit_reservations`
--

DROP TABLE IF EXISTS `customer_credit_reservations`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!50503 SET character_set_client = utf8mb4 */;
CREATE TABLE `customer_credit_reservations` (
  `customer_id` bigint NOT NULL,
  `amount` decimal(19,2) DEFAULT NULL,
  `credit_reservations_key` bigint NOT NULL,
  PRIMARY KEY (`customer_id`,`credit_reservations_key`),
  CONSTRAINT `FKnmxaefb0262tbaqt9mpogp521` FOREIGN KEY (`customer_id`) REFERENCES `customer` (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `customer_credit_reservations`
--

LOCK TABLES `customer_credit_reservations` WRITE;
/*!40000 ALTER TABLE `customer_credit_reservations` DISABLE KEYS */;
INSERT INTO `customer_credit_reservations` VALUES (2,12.34,1),(3,4.00,2);
/*!40000 ALTER TABLE `customer_credit_reservations` ENABLE KEYS */;
UNLOCK TABLES;

--
-- Table structure for table `entities`
--

DROP TABLE IF EXISTS `entities`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!50503 SET character_set_client = utf8mb4 */;
CREATE TABLE `entities` (
  `entity_type` varchar(255) NOT NULL,
  `entity_id` varchar(255) NOT NULL,
  `entity_version` longtext NOT NULL,
  PRIMARY KEY (`entity_type`,`entity_id`),
  KEY `entities_idx` (`entity_type`,`entity_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `entities`
--

LOCK TABLES `entities` WRITE;
/*!40000 ALTER TABLE `entities` DISABLE KEYS */;
/*!40000 ALTER TABLE `entities` ENABLE KEYS */;
UNLOCK TABLES;

--
-- Table structure for table `events`
--

DROP TABLE IF EXISTS `events`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!50503 SET character_set_client = utf8mb4 */;
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
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `events`
--

LOCK TABLES `events` WRITE;
/*!40000 ALTER TABLE `events` DISABLE KEYS */;
/*!40000 ALTER TABLE `events` ENABLE KEYS */;
UNLOCK TABLES;

--
-- Table structure for table `message`
--

DROP TABLE IF EXISTS `message`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!50503 SET character_set_client = utf8mb4 */;
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
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `message`
--

LOCK TABLES `message` WRITE;
/*!40000 ALTER TABLE `message` DISABLE KEYS */;
INSERT INTO `message` VALUES ('000001870d17c991-0242ac1700070000','customerService','{\"command_saga_id\":\"000001870d17c867-0242ac1700070000\",\"PARTITION_ID\":\"f6fa00bd-1d57-4528-bbfc-3d3009778f29\",\"DATE\":\"Thu, 23 Mar 2023 06:11:15 GMT\",\"b3\":\"c021acbfd705061a-bda13e42b5de5daa-1\",\"command_type\":\"io.eventuate.examples.tram.sagas.ordersandcustomers.customers.api.messaging.commands.ReserveCreditCommand\",\"command_reply_to\":\"io.eventuate.examples.tram.sagas.ordersandcustomers.orders.service.CreateOrderSaga-reply\",\"DESTINATION\":\"customerService\",\"command_saga_type\":\"io.eventuate.examples.tram.sagas.ordersandcustomers.orders.service.CreateOrderSaga\",\"command__destination\":\"customerService\",\"ID\":\"000001870d17c991-0242ac1700070000\"}','{\"orderId\":1,\"orderTotal\":{\"amount\":12.34},\"customerId\":2}',0,NULL,1679551875501),('000001870d17d854-0242ac1700080000','io.eventuate.examples.tram.sagas.ordersandcustomers.orders.service.CreateOrderSaga-reply','{\"reply_outcome-type\":\"SUCCESS\",\"commandreply_type\":\"io.eventuate.examples.tram.sagas.ordersandcustomers.customers.api.messaging.commands.ReserveCreditCommand\",\"commandreply_saga_type\":\"io.eventuate.examples.tram.sagas.ordersandcustomers.orders.service.CreateOrderSaga\",\"reply_type\":\"io.eventuate.examples.tram.sagas.ordersandcustomers.customers.api.messaging.replies.CustomerCreditReserved\",\"reply_to_message_id\":\"000001870d17c991-0242ac1700070000\",\"PARTITION_ID\":\"89a0e49c-e092-4fe0-8d1c-da9afa3ed07a\",\"commandreply_saga_id\":\"000001870d17c867-0242ac1700070000\",\"DATE\":\"Thu, 23 Mar 2023 06:11:19 GMT\",\"b3\":\"c021acbfd705061a-6f823921bd49b983-1\",\"commandreply_reply_to\":\"io.eventuate.examples.tram.sagas.ordersandcustomers.orders.service.CreateOrderSaga-reply\",\"commandreply__destination\":\"customerService\",\"DESTINATION\":\"io.eventuate.examples.tram.sagas.ordersandcustomers.orders.service.CreateOrderSaga-reply\",\"ID\":\"000001870d17d854-0242ac1700080000\"}','{}',0,NULL,1679551879296),('000001870d1d5278-0242ac1700070000','customerService','{\"command_saga_id\":\"000001870d1d5260-0242ac1700070000\",\"PARTITION_ID\":\"6aa73d13-28de-4293-a344-d38cb3ce1548\",\"DATE\":\"Thu, 23 Mar 2023 06:17:18 GMT\",\"b3\":\"64b8f274c9e6bef5-5f1bbb8e8db519e6-1\",\"command_type\":\"io.eventuate.examples.tram.sagas.ordersandcustomers.customers.api.messaging.commands.ReserveCreditCommand\",\"command_reply_to\":\"io.eventuate.examples.tram.sagas.ordersandcustomers.orders.service.CreateOrderSaga-reply\",\"DESTINATION\":\"customerService\",\"command_saga_type\":\"io.eventuate.examples.tram.sagas.ordersandcustomers.orders.service.CreateOrderSaga\",\"command__destination\":\"customerService\",\"ID\":\"000001870d1d5278-0242ac1700070000\"}','{\"orderId\":2,\"orderTotal\":{\"amount\":4},\"customerId\":3}',0,NULL,1679552238207),('000001870d1d52ec-0242ac1700080000','io.eventuate.examples.tram.sagas.ordersandcustomers.orders.service.CreateOrderSaga-reply','{\"reply_outcome-type\":\"SUCCESS\",\"commandreply_type\":\"io.eventuate.examples.tram.sagas.ordersandcustomers.customers.api.messaging.commands.ReserveCreditCommand\",\"commandreply_saga_type\":\"io.eventuate.examples.tram.sagas.ordersandcustomers.orders.service.CreateOrderSaga\",\"reply_type\":\"io.eventuate.examples.tram.sagas.ordersandcustomers.customers.api.messaging.replies.CustomerCreditReserved\",\"reply_to_message_id\":\"000001870d1d5278-0242ac1700070000\",\"PARTITION_ID\":\"1a71543a-39be-4cf8-a6db-7efdd6311841\",\"commandreply_saga_id\":\"000001870d1d5260-0242ac1700070000\",\"DATE\":\"Thu, 23 Mar 2023 06:17:18 GMT\",\"b3\":\"64b8f274c9e6bef5-4940fa4e22add5e2-1\",\"commandreply_reply_to\":\"io.eventuate.examples.tram.sagas.ordersandcustomers.orders.service.CreateOrderSaga-reply\",\"commandreply__destination\":\"customerService\",\"DESTINATION\":\"io.eventuate.examples.tram.sagas.ordersandcustomers.orders.service.CreateOrderSaga-reply\",\"ID\":\"000001870d1d52ec-0242ac1700080000\"}','{}',0,NULL,1679552238319);
/*!40000 ALTER TABLE `message` ENABLE KEYS */;
UNLOCK TABLES;

--
-- Table structure for table `offset_store`
--

DROP TABLE IF EXISTS `offset_store`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!50503 SET character_set_client = utf8mb4 */;
CREATE TABLE `offset_store` (
  `client_name` varchar(255) NOT NULL,
  `serialized_offset` longtext,
  PRIMARY KEY (`client_name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `offset_store`
--

LOCK TABLES `offset_store` WRITE;
/*!40000 ALTER TABLE `offset_store` DISABLE KEYS */;
/*!40000 ALTER TABLE `offset_store` ENABLE KEYS */;
UNLOCK TABLES;

--
-- Table structure for table `orders`
--

DROP TABLE IF EXISTS `orders`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!50503 SET character_set_client = utf8mb4 */;
CREATE TABLE `orders` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `customer_id` bigint DEFAULT NULL,
  `amount` decimal(19,2) DEFAULT NULL,
  `rejection_reason` varchar(255) DEFAULT NULL,
  `state` varchar(255) DEFAULT NULL,
  `version` bigint DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=3 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `orders`
--

LOCK TABLES `orders` WRITE;
/*!40000 ALTER TABLE `orders` DISABLE KEYS */;
INSERT INTO `orders` VALUES (1,2,12.34,NULL,'APPROVED',1),(2,3,4.00,NULL,'APPROVED',1);
/*!40000 ALTER TABLE `orders` ENABLE KEYS */;
UNLOCK TABLES;

--
-- Table structure for table `received_messages`
--

DROP TABLE IF EXISTS `received_messages`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!50503 SET character_set_client = utf8mb4 */;
CREATE TABLE `received_messages` (
  `consumer_id` varchar(255) NOT NULL,
  `message_id` varchar(255) NOT NULL,
  `creation_time` bigint DEFAULT NULL,
  `published` smallint DEFAULT '0',
  PRIMARY KEY (`consumer_id`,`message_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `received_messages`
--

LOCK TABLES `received_messages` WRITE;
/*!40000 ALTER TABLE `received_messages` DISABLE KEYS */;
INSERT INTO `received_messages` VALUES ('customerCommandDispatcher','000001870d17c991-0242ac1700070000',1679551878821,0),('customerCommandDispatcher','000001870d1d5278-0242ac1700070000',1679552238298,0),('io.eventuate.examples.tram.sagas.ordersandcustomers.orders.service.CreateOrderSaga-consumer','000001870d17d854-0242ac1700080000',1679551880082,0),('io.eventuate.examples.tram.sagas.ordersandcustomers.orders.service.CreateOrderSaga-consumer','000001870d1d52ec-0242ac1700080000',1679552238367,0);
/*!40000 ALTER TABLE `received_messages` ENABLE KEYS */;
UNLOCK TABLES;

--
-- Table structure for table `saga_instance`
--

DROP TABLE IF EXISTS `saga_instance`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!50503 SET character_set_client = utf8mb4 */;
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
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `saga_instance`
--

LOCK TABLES `saga_instance` WRITE;
/*!40000 ALTER TABLE `saga_instance` DISABLE KEYS */;
INSERT INTO `saga_instance` VALUES ('io.eventuate.examples.tram.sagas.ordersandcustomers.orders.service.CreateOrderSaga','000001870d17c867-0242ac1700070000','{\"currentlyExecuting\":0,\"compensating\":false,\"endState\":true,\"failed\":false}',NULL,1,0,0,'io.eventuate.examples.tram.sagas.ordersandcustomers.orders.api.messaging.sagas.createorder.CreateOrderSagaData','{\"orderDetails\":{\"customerId\":2,\"orderTotal\":{\"amount\":12.34}},\"orderId\":1}'),('io.eventuate.examples.tram.sagas.ordersandcustomers.orders.service.CreateOrderSaga','000001870d1d5260-0242ac1700070000','{\"currentlyExecuting\":0,\"compensating\":false,\"endState\":true,\"failed\":false}',NULL,1,0,0,'io.eventuate.examples.tram.sagas.ordersandcustomers.orders.api.messaging.sagas.createorder.CreateOrderSagaData','{\"orderDetails\":{\"customerId\":3,\"orderTotal\":{\"amount\":4}},\"orderId\":2}');
/*!40000 ALTER TABLE `saga_instance` ENABLE KEYS */;
UNLOCK TABLES;

--
-- Table structure for table `saga_instance_participants`
--

DROP TABLE IF EXISTS `saga_instance_participants`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!50503 SET character_set_client = utf8mb4 */;
CREATE TABLE `saga_instance_participants` (
  `saga_type` varchar(255) NOT NULL,
  `saga_id` varchar(100) NOT NULL,
  `destination` varchar(100) NOT NULL,
  `resource` varchar(100) NOT NULL,
  PRIMARY KEY (`saga_type`,`saga_id`,`destination`,`resource`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `saga_instance_participants`
--

LOCK TABLES `saga_instance_participants` WRITE;
/*!40000 ALTER TABLE `saga_instance_participants` DISABLE KEYS */;
/*!40000 ALTER TABLE `saga_instance_participants` ENABLE KEYS */;
UNLOCK TABLES;

--
-- Table structure for table `saga_lock_table`
--

DROP TABLE IF EXISTS `saga_lock_table`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!50503 SET character_set_client = utf8mb4 */;
CREATE TABLE `saga_lock_table` (
  `target` varchar(100) NOT NULL,
  `saga_type` varchar(255) NOT NULL,
  `saga_Id` varchar(100) NOT NULL,
  PRIMARY KEY (`target`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `saga_lock_table`
--

LOCK TABLES `saga_lock_table` WRITE;
/*!40000 ALTER TABLE `saga_lock_table` DISABLE KEYS */;
/*!40000 ALTER TABLE `saga_lock_table` ENABLE KEYS */;
UNLOCK TABLES;

--
-- Table structure for table `saga_stash_table`
--

DROP TABLE IF EXISTS `saga_stash_table`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!50503 SET character_set_client = utf8mb4 */;
CREATE TABLE `saga_stash_table` (
  `message_id` varchar(100) NOT NULL,
  `target` varchar(100) NOT NULL,
  `saga_type` varchar(255) NOT NULL,
  `saga_id` varchar(100) NOT NULL,
  `message_headers` varchar(1000) NOT NULL,
  `message_payload` varchar(1000) NOT NULL,
  PRIMARY KEY (`message_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `saga_stash_table`
--

LOCK TABLES `saga_stash_table` WRITE;
/*!40000 ALTER TABLE `saga_stash_table` DISABLE KEYS */;
/*!40000 ALTER TABLE `saga_stash_table` ENABLE KEYS */;
UNLOCK TABLES;

--
-- Table structure for table `snapshots`
--

DROP TABLE IF EXISTS `snapshots`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!50503 SET character_set_client = utf8mb4 */;
CREATE TABLE `snapshots` (
  `entity_type` varchar(255) NOT NULL,
  `entity_id` varchar(255) NOT NULL,
  `entity_version` varchar(255) NOT NULL,
  `snapshot_type` longtext NOT NULL,
  `snapshot_json` longtext NOT NULL,
  `triggering_events` longtext,
  PRIMARY KEY (`entity_type`,`entity_id`,`entity_version`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `snapshots`
--

LOCK TABLES `snapshots` WRITE;
/*!40000 ALTER TABLE `snapshots` DISABLE KEYS */;
/*!40000 ALTER TABLE `snapshots` ENABLE KEYS */;
UNLOCK TABLES;
/*!40103 SET TIME_ZONE=@OLD_TIME_ZONE */;

/*!40101 SET SQL_MODE=@OLD_SQL_MODE */;
/*!40014 SET FOREIGN_KEY_CHECKS=@OLD_FOREIGN_KEY_CHECKS */;
/*!40014 SET UNIQUE_CHECKS=@OLD_UNIQUE_CHECKS */;
/*!40101 SET CHARACTER_SET_CLIENT=@OLD_CHARACTER_SET_CLIENT */;
/*!40101 SET CHARACTER_SET_RESULTS=@OLD_CHARACTER_SET_RESULTS */;
/*!40101 SET COLLATION_CONNECTION=@OLD_COLLATION_CONNECTION */;
/*!40111 SET SQL_NOTES=@OLD_SQL_NOTES */;

-- Dump completed on 2023-03-23  6:20:40
```
### (7). 总结
按照官网那张图的理解,业务系统(order/customer)会通过事务表的方式保存事件后,工作就已结束了,理当交给cdc来操作,从现在的剖析来看,好像业务系统强依赖了Kafka,没有kafka就启动不了. 