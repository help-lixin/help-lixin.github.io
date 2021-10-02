---
layout: post
title: 'Kafka总结' 
date: 2021-10-01
author: 李新
tags:  Kafka
---

### (1). 概述
三年前看过RocketMQ的源码,以至于,每次选择MQ都会不加思索的建议使用RocketMQ.     
最近要开始看Debezium的源码,发现它与Kakfa耦合太紧,所以,在看源码之前,先要大概的了解Kafa,结果:发现Kakfa的生态圈比RocketMQ要全很多.     
所以,有时候:固定你思维的是你现在的认知和知识.   
### (2). Kafka介绍
Kafka是Apache项目下的一个分布式流处理平台,它的流行是因为系统的设计和操作简单,能充分利用磁盘的顺序读写特性.kafka每秒钟能有百万条消息的吞吐量,因此很适合实时的数据流处理.   

### (3). Kafka基本概念介绍
+ Broker
  - 运行的kafka节点
+ Topic
  - 主题,主题中的每条消息包括key-value和timestamp.可以定义多个topic,每个topic又可以划分为多个分区.  
+ Partition
  - topic下的消息分区,通过key取哈希后把消息映射分发到一个指定的分区,每个分区都映射到broker上的一个目录.一般每个分区存储在一个broker上.  
+ Producer
  - 消息生产者
+ Consumer
  - 消息消费者
+ Consumer Group
  - 消息消费组,同一个消费组,只能有一个consumer能消费消息.
+ Replica
  - 副本,每个分区按照生产者的消息达到顺序存放.每个分区副本都有一个leader.
+ Leader Replica
  - Leader角色的分区副本,Leader角色的分区处理消息的读写请求.
+ Follower Replica
  - Follower角色的分区副本,负责从Leader拉取数据到本地,实现分区副本的创建.  
+ Zookeeper
  - Kafka的选举以及元数据信息是依赖于ZK的.

### (4). Kafka学习目录
["Kafka单机安装(一)"](/2021/09/13/Kafka-Install.html)  
["Kafka集群安装(二)"](/2021/10/01/Kafka-Cluster-Install.html)   
["Kafka常用命令(三)"](/2021/10/01/Kafka-Command.html)     
["Kafka生产者配置和案例(四"](/2021/10/01/Kafka-Producer.html)     
["Kafka消费者案例(五)"](/2021/10/01/Kafka-Consumer.html)           
["Spring Boot与Kafka集成(六)"](/2021/10/01/Kafka-SpringBoot.html)          
["Spring Boot与Kafka集成源码之KafkaProperties(七)"](/2021/10/01/Kafka-SpringBoot-KafkaProperties.html)   
["Spring Boot与Kafka集成源码之@KafkaListener(八)"](/2021/10/01/Kafka-SpringBoot-KafkaListener.html)   
["Spring Boot与Kafka集成源码之ConcurrentMessageListenerContainer(九)"](/2021/10/01/Kafka-SpringBoot-ConcurrentMessageListenerContainer.html)   