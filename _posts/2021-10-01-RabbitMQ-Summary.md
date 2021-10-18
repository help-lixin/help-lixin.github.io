---
layout: post
title: 'RabbitMQ总结' 
date: 2021-10-01
author: 李新
tags:  RabbitMQ
---

### (1). 概述
大部份的MQ基本上都了解差不多了,由于,现在手上有一个项目有用到RabbitMQ,所以,对RabbitMQ进行一个深入的了解(以及与Spring Cloud Stream的集成).   
### (2). 学习目录
+ ["RabbitMQ架构以及基本概念(一)"](/2021/10/18/RabbitMQ-Architecture.html)   
+ ["RabbitMQ Docker单机安装(二)"](/2021/10/18/RabbitMQ-Docker-Install.html)   
### (3). 总结
感觉RabbitMQ的主要亮点是:消息路由机制,对于消息在存储模型上,最终是通过Queue来承载(与Kafka和RocketMQ完全相反),假如一个Queue每天有千万级的消息,可想而知,RabbitMQ的吞吐量是会有所下降的.   
此处有一个疑问:RabbitMQ里,当消费只有一个Queue的时候,多线程消费时,ack是如何保证的(即:Kakfa Rebalance机制).    
我暂时理解的结论是:RabbitMQ不适合量比较大的消息,反而适合那种消费量小,并且,需要对消息进行区分的队列(路由是它最大的亮点).    