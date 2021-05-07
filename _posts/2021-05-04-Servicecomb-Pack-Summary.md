---
layout: post
title: 'Servicecomb-Pack是什么(一)'
date: 2021-05-04
author: 李新
tags:  Servicecomb-Pack
---

### (1). 为什么要研究Servicecomb Pack
> 阿里开源了一套分布式事务解决方案(Seata),AT模式有着它的优点.  
> 但是,有些场景下TCC和Saga模式会更加的适合,Seta虽然有TCC和Saga的解决方案,可是,看源码后,明显能感觉到阿里着重点是在AT模式,TCC和Saga支持较少.  
> 所以,才会好奇看下:Servicecomb Pack.    

### (2). Servicecomb Pack是什么?
> Apache ServiceComb Pack 是由华为开源的一个微服务应用的"数据最终一致性解决方案"(分布式事务解决方案).      
> ["Apache ServiceComb Pack"](https://github.com/apache/servicecomb-pack)   

### (3). Servicecomb Pack架构
> ServiceComb Pack 架构是由 alpha 和 omega组成,其中:  
> alpha充当协调者的角色,主要负责对事务进行管理和协调(Transaction Coordinator).   
> omega是微服务中内嵌的一个agent,负责对调用请求进行拦截并向alpha上报事务事件(Transaction Manager).  
> 在命名上,不太理解:alpha和omega,不过无所谓了,分布式事务的思想,反正都差不多,只是命名上的差异而已.   

!["ServiceComb Pack 架构"](/assets/servicecomb-pack/imgs/ServiceComb-Pack-Architecture.png)
### (4). Servicecomb Pack学习目录
> 
### (5). Servicecomb Pack缺点
> Servicecomb Pack的文档少得有点可怜,很多东西要靠自己摸索.
