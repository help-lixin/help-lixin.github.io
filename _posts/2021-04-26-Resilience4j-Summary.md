---
layout: post
title: 'Resilience4j 介绍(一)'
date: 2021-04-26
author: 李新
tags:  Resilience4j
---

### (1). Resilience4j是什么?
> Resilience4j是一款轻量级,易于使用的容错库,其灵感来自于Netflix Hystrix,但是专为Java 8和函数式编程而设计.轻量级,因为库只使用了Vavr,它没有任何其他外部依赖下.相比之下,Netflix Hystrix对Archaius具有编译依赖性,Archaius具有更多的外部库依赖性,例如Guava和Apache Commons Configuration.   
> 要使用Resilience4j,不需要引入所有依赖,只需要选择你需要的. 

### (2). Resilience4j 核心模块
* resilience4j-circuitbreaker: Circuit breaking[断路器]   
* resilience4j-ratelimiter: Rate limiting[限流]  
* resilience4j-bulkhead: Bulkheading[基于信号量的隔离]   
* resilience4j-retry: Automatic retrying (sync and async)[请求重试]  
* resilience4j-cache: Result caching[缓存]   
* resilience4j-timelimiter: Timeout handling[限时]   

### (3). 学习目录
> ["Resilience4j Circuit Breaker(二)"](/2021/04/25/Resilience4j-Circuit-Breaker.html)   
> 
> 
### (4). 总结
> 