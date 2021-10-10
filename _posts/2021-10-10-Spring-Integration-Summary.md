---
layout: post
title: 'Spring Integration总结' 
date: 2021-10-10
author: 李新
tags:  SpringIntegration
---

### (1). Spring Integration 是什么
Spring Integration对Spring编程模型进行了扩展,使得能够支持著名的“企业集成模式”.通过SI(Spring Integration)可以在基于Spring的应用中引入轻量级的“消息驱动模式”,并且支持“通过声明式的适配器”与外部系统进行集成.这些“适配器”相较于Spring对于“remoting（远程调用）”、“messaging（事件消息）”、“scheduling（任务调度）”方面的支持,提供了更高层次的一种抽象.   
SI的首要目标是:为“构建企业集成方案、维护系统间通信”提供一种简单模型,应用该模型所产出的代码是可维护、可测试的.   

### (2). Spring Integration的特性
+ 实现大多数企业集成模式
+ 端点Endpoint
+ 通道Channel (点对点和发布/订阅)
+ 聚合器Aggregator
+ 过滤器Filter
+ 转换器Transformer
+ 控制总线
+ 与外部系统集成
+ ReST / HTTP
+ FTP / SFTP
+ TCP / UDP
+ ... ...

### (3). Spring Integration学习目录
+ ["Spring Integration概念与介绍(一)"](/2021/10/06/Spring-Integration-Introduce.html)          
+ ["Spring Integration Hello World入门案例(二)"](/2021/10/06/Spring-Integration-Hello-World.html)        
+ ["查看Spring Integration相关业务模型能力(三)"](/2021/10/09/Spring-Integration-Bussiness-Modeling.html)    
+ ["Spring Integration源码之IntegrationNamespaceHandler(四)"](/2021/10/06/Spring-Integration-IntegrationNamespaceHandler.html)   
+ ["Spring Integration源码之PointToPointChannelParser(五)"](/2021/10/09/Spring-Integration-PointToPointChannelParser.html)   
+ ["Spring Integration源码之ServiceActivatorParser(六)"](/2021/10/09/Spring-Integration-ServiceActivatorParser.html)   
+ ["Spring Integration源码之ServiceActivatorFactoryBean(七)"](/2021/10/09/Spring-Integration-ServiceActivatorFactoryBean.html)    
+ ["Spring Integration源码之ServiceActivatingHandler(八)"](/2021/10/09/Spring-Integration-ServiceActivatingHandler.html)    
+ ["Spring Integration源码之ConsumerEndpointFactoryBean(九)"](/2021/10/09/Spring-Integration-ConsumerEndpointFactoryBean.html)   
