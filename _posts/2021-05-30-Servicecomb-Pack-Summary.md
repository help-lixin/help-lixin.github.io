---
layout: post
title: 'Servicecomb Pack是什么(一)'
date: 2021-05-30
author: 李新
tags:  Servicecomb-Pack
---

### (1). 为什么要研究Servicecomb Pack
> 阿里开源了一套分布式事务解决方案(Seata),通过阅读完源码后,发现:阿里针对分布式事务的着重点是在AT模式,而TCC和Saga支持不足.  
> 所以,才会想看下:Servicecomb Pack,是否有对这些不足进行解决(<font color='red'>TCC模式下的幂等/悬空/...</font>).       

### (2). Servicecomb Pack是什么?
> Apache ServiceComb Pack 是由华为开源的一个微服务应用的<font color='red'>数据最终一致性解决方案(分布式事务解决方案).</font>      
> ["Apache ServiceComb Pack"](https://github.com/apache/servicecomb-pack)   

### (3). Servicecomb Pack架构
> ServiceComb Pack 架构是由 alpha 和 omega组成,其中:  
> alpha充当协调者的角色,主要负责对事务进行管理和协调(类似于:Transaction Coordinator).   
> omega是微服务中内嵌的一个agent,负责对调用请求进行拦截并向alpha上报事务事件(类似于:Transaction Manager).  

!["ServiceComb Pack 架构"](/assets/servicecomb-pack/imgs/ServiceComb-Pack-Architecture.png)

### (4). Servicecomb Pack处理流程图
> 注意:失败时是由omega向alpha进行汇报即可,但是,补偿操作(compensate)是由:alpha向所有的:omega发出的指令.   

> Saga处理流程图:    
> Saga成功处理流程图:
!["Saga成功处理流程图"](/assets/servicecomb-pack/imgs/saga-successful_scenario.png)    

> Saga失败处理流程图   
!["Saga失败处理流程图"](/assets/servicecomb-pack/imgs/saga-exception_scenario.png)    

> Saga超时处理流程图   
!["Saga超时处理流程图"](/assets/servicecomb-pack/imgs/saga-timeout_scenario.png)   


> TCC处理流程图:  

>TCC 正常处理流程图   
!["TCC 正常处理流程图"](/assets/servicecomb-pack/imgs/successful_scenario_TCC.png)   

> TCC 异常处理流程图
!["TCC 异常处理流程图"](/assets/servicecomb-pack/imgs/exception_scenario_TCC.png)    
### (5). Servicecomb Pack学习目录
> ["Servicecomb Pack之Saga+Docker环境搭建(二)"](/2021/05/04/Servicecomb-Saga-Docker.html)   
> ["Servicecomb Pack之Saga本地环境搭建入门(三)"](/2021/05/04/Servicecomb-Saga.html)  

### (6). Servicecomb Pack缺点
> Servicecomb Pack的文档少得有点可怜,很多东西要靠自己摸索. 