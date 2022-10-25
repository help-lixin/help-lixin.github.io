---
layout: post
title: 'Zeebe总结' 
date: 2022-10-14
author: 李新
tags:  Zeebe
---


### (1). 概述
传统工作流的缺陷: 
+ 传统的工作流引擎,编排的大部分是人工审批任务,意味着任务流转效率低,系统吞吐低.而当下微服务大部分是程序化的自动任务,意味着任务高效流转,系统吞吐高.单点架构、同步响应、高度依赖DB的Activiti，显然支撑不了这样的场景.   
+ 传统的工作流引擎,都以jar包的形式,嵌入到业务程序中,直接通过调用本地方法的方式调度起业务TaskHandler.在单体架构下,这种集成方式简单易用.但是在微服务架构下,工作流的任务往往是分布在多个服务的,而且同一个服务往往还会根据负载情况部署不同数量的实例.如果还是采用引擎主动调用的方式,怎么寻址到具体的TaskHandler?当后端业务服务处理能力本身是瓶颈的时候,如果引擎还是不断的调用,只会进一步压垮服务.   

### (2). Zeebe架构图
!["Zeebe架构"](/assets/zeebe/imgs/zeebe-architecture.png)   

### (3). Zeebe架构
+ Client(Job Workers),客户端是嵌入到应用程序(执行业务逻辑的微服务)的库,主要用于跟Zeebe集群连接通信.   
  - 发布工作流.  
  - 执行业务逻辑.      
  - 创建工作流实例/发布消息/激活任务/完成任务/失败任务.   
  - 处理运维问题.   
  - 更新实例流程变量.    
+ Gateway   
  - 转发请求到Brokers   
  - Gateway是无状态(stateless)无会话(sessionless)的,可以按需增加节点,以负载均衡及高可用.   
+ Broker   
  - 处理客户端发送的指令.   
  - 存储和管理运行中流程实例的状态.  
  - 分配任务给Job Workers(Client).   
+ Exporter   
  - 监控当前运行流程实例的状态.   
  - 分析历史的工作流数据以做审计或BI.   
  - 跟踪Zeebe抛出的异常(incident).   

### (4). Zeebe基础以及Client SDK源码剖析
+ ["Zeebe源码编译(一)"](/2022/02/02/Zeebe-Source-Compile.html)  
+ ["Zeebe Broker安装(二)"](/2022/02/02/Zeebe-Broker-Install.html)   
+ ["Zeebe Client简单入门(三)"](/2022/02/02/Zeebe-Client-HelloWorld.html)  
+ ["Zeebe ZeebeClient源码部析(四)"](/2022/02/02/Zeebe-ZeebeClient.html)  
+ ["Zeebe集群搭建(五)"](/2022/09/14/Zeebe-Cluster.html)  
+ ["Zeebe源码之ZeebeClientFutureImpl(六)"](/2022/09/15/Zeebe-ZeebeClientFutureImpl.html)   

### (5). Zeebe gateway目录
+ ["Zeebe Gateway源码之StandaloneGateway创建AtomixCluster详解(一)"](/2022/09/16/Zeebe-StandaloneGateway-AtomixCluster.html)  
+ ["Zeebe Gateway源码之StandaloneGateway创建ActorScheduler详解(二)"](/2022/09/16/Zeebe-StandaloneGateway-ActorScheduler.html) 
+ ["Zeebe Gateway源码之StandaloneGateway创建BrokerClient详解(三)"](/2022/09/16/Zeebe-StandaloneGateway-BrokerClient.html) 
+ ["Zeebe Gateway源码之StandaloneGateway创建Gateway详解(四)"](/2022/09/16/Zeebe-StandaloneGateway-Gateway.html)  

### (6). Zeebe broker目录
+ ["Zeebe源码之Broker初始化(一)"](/2022/10/14/Zeebe-Broker.html)  
+ ["Zeebe源码之BrokerStartupProcess(二)"](/2022/10/14/Zeebe-BrokerStartupProcess.html)   
+ ["Zeebe源码之StartupStep(三)"](/2022/10/14/Zeebe-StartupStep.html)   
+ ["Zeebe ClusterServicesStep源码之NettyMessagingService初始化之Client(四)"](/2022/10/14/Zeebe-NettyMessagingService-Client-Init.html)  
+ ["Zeebe ClusterServicesStep源码之NettyMessagingService初始化之Server(五)"](/2022/10/14/Zeebe-NettyMessagingService-Server-Init.html)   
+ ["Zeebe ClusterServicesStep源码之NettyUnicastService(六)"](/2022/10/14/Zeebe-NettyUnicastService.html)  
+ ["Zeebe ClusterServicesStep源码之NodeDiscoveryProvider(七)"](/2022/10/14/Zeebe-NodeDiscoveryProvider.html)  
+ ["Zeebe ClusterServicesStep源码之GroupMembershipProtocol(八)"](/2022/10/14/Zeebe-GroupMembershipProtocol.html)  
+ ["Zeebe ClusterServicesStep源码之ClusterCommunicationService(九)"](/2022/10/14/Zeebe/ClusterCommunicationService.html)  
+ ["Zeebe ClusterServicesStep源码之ClusterEventService(十)"](/2022/10/14/Zeebe-ClusterEventService.html)   
+ ["Zeebe ClusterServicesStep源码之ApiMessagingServiceStep(十一)"](/2022/10/14/Zeebe-ApiMessagingServiceStep.html)  
+ ["Zeebe ClusterServicesStep源码之NettyMessagingService案例(十二)"](/2022/10/14/Zeebe-NettyMessagingService-Example.html)  
+ ["Zeebe ClusterServicesStep源码之RemoteServerConnection(十三)"](/2022/10/14/Zeebe-RemoteServerConnection.html)  
+ ["Zeebe ClusterServicesStep源码之HandlerRegistry(十四)"](/2022/10/14/Zeebe-HandlerRegistry.html)  
+ ["Zeebe ClusterServicesStep源码之CommandApiRequestHandler(十五)"](/2022/10/14/Zeebe-CommandApiRequestHandler.html)  
