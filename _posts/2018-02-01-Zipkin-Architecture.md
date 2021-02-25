---
layout: post
title: 'Zipkin 架构分析(二)'
date: 2018-02-01
author: 李新
tags: Zipkin
---

### (1). 先看下Zipkin架构图
!["zipkin architecture"](/assets/zipkin/imgs/zipkin-architecture.png)

> Instrumented server(Reporter):可以理解为agent,它和业务进程捆绑在一起.    
> Transport:传输层.     
> Collector:收集器(可以理解为Controller).  
> Store:存储层(MySQL/ES/...).    
> API: 提供接口给外部(UI)访问.   
> <font color='red'>总结:Reporter通过Transport向Collector(Zipkin Server)汇报数据,然后,Store存储.而API+UI负责做展现.</font>  

### (3). ZipKin与Java整合
> 1. 方案一:zipkin(zipkin-reporter/zipkin-sender-okhttp3),向zipkin server汇报数据.   
> 2. 方案二:barve(brave),向zipkin server汇报数据.   
> 3. 方案三:spring-cloud-starter-sleuth,向zipkin server汇报数据.   
> 4. 这三个方案的关系是什么呢?spring-cloud-starter-sleuth对barve进行了整合.而brave又对zipkin进行了API的包装.   

### (4). 研究步骤
> 1. brave.   
> 2. spring-cloud-starter-sleuth.   
> 3. 原因:先了解了API,对于:spring-cloud-starter-sleuth只是一个整合而已.  

### (5). zipkin类图和执行流程图
!["brave类图"](/assets/zipkin/imgs/zipkin-Class-Diagram.jpg)
!["brave执行流程图"](/assets/zipkin/imgs/zipkin-Sequence-Diagram.jpg)

### (6). 总结
> 从执行流程图来看,好像也没什么好讲的啦!就是那么的so easy. 