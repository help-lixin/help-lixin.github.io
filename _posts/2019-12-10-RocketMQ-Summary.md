---
layout: post
title: 'RocketMQ总结' 
date: 2023-01-23
author: 李新
tags:  RocketMQ
---

### (1). 前言
公司的业务对于MQ有一些自己的业务特性,所以,为了解决当下问题,临时自研了一套MQ(Redis),因为,Redis是内存级的,所以,一直想探索其它的MQ看能否符合公司的业务,选择RocketMQ是因为纯Java代码,能Hold住源码进行改造,所以,对RocketMQ的源码剖析,不会是纯用SDK,而是想法绕过现有的SDK,直接通过RPC访问,并剖析源码.  

### (2). Consumer
+ ["RocketMQ源码之DefaultMQPullConsumer介绍(一)"](/2019/12/10/RocketMQ-DefaultMQPullConsumer.html) 
+ ["RocketMQ源码之根据主题或Key查询消息(二)"](/2019/12/10/RocketMQ-QueryMessage.html) 
+ ["RocketMQ源码之拉取消息(三)"](/2019/12/10/RocketMQ-PullMessage.html)

### (3). Producer
+ ["RocketMQ源码之DefaultMQProducer介绍(一)"](/2019/12/10/RocketMQ-DefaultMQProducer.html)  
+ ["RocketMQ源码之生产消息(二)"](/2019/12/10/RocketMQ-SendMessage.html)  

### (4). Broker
+["RocketMQ Broker存储层介绍(一)"](/2019/12/10/RocketMQ-Broker-Concept.html)   