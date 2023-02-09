---
layout: post
title: 'Spring Message图解' 
date: 2023-02-08
author: 李新
tags:  SpringMessage
---

### (1). 概述

最近研究Pulsar时,发现,它好像没有与Spring Boot集成的start,有一个想法,想自己来实现这个功能,
不过,需要透明化一些,哪天企业决策切换成:RocketMQ/Kafka,不能变动任何代码,所以,自己写这个集成,最好是向:Spring Message靠拢.  

### (2). Spring Message发送消息接口定义UML

!["Spring Message发送消息定义UML"](/assets/spring-message/imgs/MessageSending.jpg)

### (3). Spring Message接受消息接口定义UML
> 通过接口就能看出来,发送消息很简单,复杂的是:接受消息,因为,接受消息涉及再次入MQ,以及响应(看RocketMQ源码时,发现它并没有遵循这个).  


!["Spring Message接受消息定义UML"](/assets/spring-message/imgs/MessageReceiving.jpg)

### (4). Spring Message Channel定义UML
> MessageChannel是最终消息发送的通道. 

!["Spring Message Channel定义UML"](/assets/spring-message/imgs/MessageChannel.jpg)

### (5). 总结
遵循上面接口开发即可,不过Pulsar对于生产和消费有很多的配置项,是需要注意,不能因为封装,把特性给阉割. 
