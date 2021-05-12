---
layout: post
title: 'ShedLock 简介(一)'
date: 2021-05-15
author: 李新
tags:  ShedLock
---

### (1). ShedLock是什么
> ShedLock是一个在分布式环境中使用的定时任务框架,用于解决在分布式环境中的多个实例的相同定时任务在同一时间点重复执行的问题.    
> 解决思路是:通过对公用的数据库中的某个表进行记录和加锁,使得同一时间点只有第一个执行定时任务并成功在数据库表中写入相应记录的节点能够成功执行而其他节点直接跳过该任务.  
> 当然不只是支持数据库,目前已经实现的支持数据存储类型除了经典的关系型数据库,还包括MongoDB,Zookeeper,Redis,Hazelcast,ElasticSearch,Cassandra,Consul,Etcd... 

### (2). 为什么要用ShedLock
> 其实,自己做分布锁的框架也是可行的,但是,已经有开源的了,不论是节约时间,还是,成本来说,建议用开源的.
> <font color='red'>ShedLock的业务模型,做得还是挺好的,针对分布式锁提供者(MySQL/Redis...),可以自由切换.</font>  
> 只是,我会比较好奇是否存在Bug(比如:Redis做锁),期待后面的源码探索.   

### (3). 学习目录
> ["ShedLock 入门程序(二)"](/2021/05/11/ShedLock-HelloWorld.html)   
