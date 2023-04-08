---
layout: post
title: 'Eventuate源码剖析' 
date: 2023-04-08
author: 李新
tags:  Eventuate
---

### (1). 背景
最近在研究Eventuate,该开源框架出自名师之手:Chris Richardson(非常典型的一本:微服务架构设计模式),而,对于一个框架的引用,不能简简单的停留在使用阶段,而应该要知道其边界,知道其能力,所以,我一般除了会用框架外,会把框架的源码大体的从头到尾看一遍.  

### (2). UML图
!["Common"](/assets/eventuate/imgs/Eventuate_Model_Common_0.jpg)
!["Consumer"](/assets/eventuate/imgs/Eventuate_Model_Consumer_5.jpg)
!["Sql方言定义"](/assets/eventuate/imgs/Eventuate_Model_EventuateSqlDialect_2.jpg)
!["Id生成策略"](/assets/eventuate/imgs/Eventuate_Model_IdGenerator_3.jpg)
!["消息解码器"](/assets/eventuate/imgs/Eventuate_Model_MessageHandlerDecorator_1.jpg)
!["生产者"](/assets/eventuate/imgs/Eventuate_Model_Producer_4.jpg)
!["SagaDSL定义"](/assets/eventuate/imgs/Eventuate_Model_SagaDSL_8.jpg)
!["SagaLock管理"](/assets/eventuate/imgs/Eventuate_Model_SagaLockManager_6.jpg)
!["Saga通用"](/assets/eventuate/imgs/Eventuate_Model_Saga_7.jpg)

### (3). 总结
由于最近项目很赶时间,没办法一边看源码,一边写文档了,只能先在这里把框架的大体UML图贴出来. 