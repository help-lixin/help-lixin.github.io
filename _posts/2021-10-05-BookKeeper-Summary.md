---
layout: post
title: 'Apache BookKeeper总结' 
date: 2021-10-05
author: 李新
tags:  BookKeeper
---

### (1). Apache BookKeeper背景介绍
BookKeeper的开发者Benjamin Reed、Flavio Junqueira、Ivan Kelly凭借搭建ZooKeeper的经验设计了一个灵活的系统,能够支持多种工作负载.   
最初,BookKeeper是分布式系统的预写式日志(WAL)机制.现在BookKeeper已经发展成为支持多个企业级系统的基础构建模块,如: Twitter的EventBus、雅虎的Apache Pulsar等.    

### (2). Apache BookKeeper是什么
BookKeeper是一种优化实时工作负载的存储服务,具有可扩展、高容错、低延迟的特点.通常,企业级的实时存储平台应符合以下几项要求:
+ 以极低的延迟(小于 5 毫秒)读写entry流.  
+ 能够持久、一致、容错地存储数据.  
+ 在写数据时,能够进行流式传输或追尾传输.  
+ 有效地存储、访问历史数据与实时数据

### (3). Apache BookKeeper架构图

!["Apache BookKeeper架构图"](/assets/bookkeeper/imgs/bookkeeper.png)

### (4). Apache BookKeeper的概念及术语
+ Cluster
  - 所有的Bookie组成一个集群(连接到同一个zk地址的Bookie集合).  
+ Bookie
  - BookKeeper的存储节点,也即Server节点.   
+ Ledger
  - Ledger是对一个log文件的抽象,它本质上是由一系列Entry(类似于Kafka的msg)组成的,client在向BookKeeper写数据时是交给Ledger中写的.  
+ Entry
  - Entry本质上就是一条数据,它会有一个id做标识.   
+ Journal
  - Write ahead log(WAL),也就是数据是先写到Journal中.  
+ Ensemble
  - Set of Bookies across which a ledger is striped,一个Ledger所涉及的Bookie集合,初始化Ledger时,需要指定这个Ledger可以在几台Bookie上存储.  
+ Write Quorum Size
  - Number of replicas,要写入的副本数.
+ Ack Quorum Size
  - Number of responses needed before client’s write is satisfied,当这么多副本写入成功后才会向client返回成功.
  - 比如副本数设置了3,这个设置了2,client会同时向三副本写入数据,当收到两个成功响应后,会认为数据已经写入成功.  
+ LAC
  - Last Add Confirmed,Ledger中已经确认的最近一条数据的entry id.
### (5). Apache BookKeeper学习目录

