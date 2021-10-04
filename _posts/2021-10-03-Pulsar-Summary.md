---
layout: post
title: 'Pulsar总结' 
date: 2021-10-03
author: 李新
tags:  Pulsar
---

### (1). Pulsar是什么
Pulsar是一个用于服务器到服务器的消息系统,具有多租户、高性能等优势.      
Pulsar最初由Yahoo开发,目前由Apache软件基金会管理.      
Pulsar 的关键特性如下:  

+ Pulsar的单个实例原生支持多个集群,可跨机房在集群间无缝地完成消息复制.   
+ 极低的发布延迟和端到端延迟.  
+ 无缝扩展到超过一百万个topic.   
+ 简单的客户端 API,支持 Java、Go、Python 和 C++   
+ 支持多种topic订阅模式(独占订阅、共享订阅、故障转移订阅)   
+ 通过Apache BookKeeper提供的持久化消息存储机制保证消息传递.   
  - 由轻量级的serverless计算框架Pulsar Functions实现流原生的数据处理.   
  - 基于Pulsar Functions的serverless connector框架Pulsar IO使得数据更易移入、移出 Apache Pulsar.  
  - 分层式存储可在数据陈旧时,将数据从热存储卸载到冷/长期存储(如S3、GCS)中.  


### (2). Pulsar架构
+ Brokers
  - Pulsar的Broker是一个无状态组件,主要负责运行另外的两个组件(REST API/调度分发器)
+ Apache ZooKeeper
  - Pulsar使用Apache ZooKeeper进行元数据存储、集群配置和协调
+ Apache BookKeeper
  - Pulsar用Apache BookKeeper作为持久化存储.BookKeeper是一个分布式的预写日志(WAL)系统.   
+ Pulsar Proxy
  - Pulsar客户端和Pulsar集群交互的一种方式就是直连Pulsar brokers.然而,在某些情况下,这种直连既不可行也不可取,因为客户端并不知道broker的地址.例如:云环境.   

!["Pulsar架构图"](/assets/pulsar/imgs/pulsar-system-architecture.png)

### (3). Pulsar学习目录
["Pulsar集群搭建(一)"](/2021/10/03/Pulsar-Cluster-Install.html)
