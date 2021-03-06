---
layout: post
title: 'Servicecomb Pack是什么(一)'
date: 2021-05-11
author: 李新
tags:  Servicecomb-Pack
---

### (1). 为什么要研究Servicecomb Pack
> 阿里开源了一套分布式事务解决方案(Seata),通过阅读完源码后,发现:阿里针对分布式事务的着重点是在AT模式,而TCC和Saga支持没有AT模式那么多.  
> 所以,才会想看下:Servicecomb Pack,是否有对这些不足进行解决(<font color='red'>TCC模式下的幂等/悬挂/...</font>).       
> <font color='red'>看完一部份源码后,回来做个小总结:Servicecomb Pack有严重的Bug,不能适用于生存环境,可以跳到最后看我写的总结!</font>   

### (2). Servicecomb Pack是什么?
> Apache ServiceComb Pack 是由华为开源的一个微服务应用的<font color='red'>数据最终一致性解决方案(分布式事务解决方案).</font>      
> ["Apache ServiceComb Pack"](https://github.com/apache/servicecomb-pack)   

### (3). Servicecomb Pack架构
> ServiceComb Pack 架构是由 alpha 和 omega组成,其中:  
> alpha充当协调者的角色,主要负责对事务进行管理和协调(类似于:Transaction Coordinator).   
> omega是微服务中内嵌的一个agent,负责对调用请求进行拦截并向alpha上报事务事件(类似于:Transaction Manager).  

!["ServiceComb Pack 架构"](/assets/servicecomb-pack/imgs/ServiceComb-Pack-Architecture.png)

### (4). Servicecomb Pack 内部模块架构图
!["Servicecomb Pack 内部模块架构图"](/assets/servicecomb-pack/imgs/image-pack-system-archecture.png)

### (5). Servicecomb Pack处理流程图
> <font color='red'>注意:失败时是由omega向alpha进行汇报即可,但是,补偿操作(compensate)是由:alpha向所有的:omega发出的指令(感觉和Seata一样).</font>   

> Saga成功处理流程图:
!["Saga成功处理流程图"](/assets/servicecomb-pack/imgs/saga-successful_scenario.png)    

> Saga失败处理流程图   
!["Saga失败处理流程图"](/assets/servicecomb-pack/imgs/saga-exception_scenario.png)    

> Saga超时处理流程图   
!["Saga超时处理流程图"](/assets/servicecomb-pack/imgs/saga-timeout_scenario.png)   


>TCC 正常处理流程图   
!["TCC 正常处理流程图"](/assets/servicecomb-pack/imgs/successful_scenario_TCC.png)   

> TCC 异常处理流程图
!["TCC 异常处理流程图"](/assets/servicecomb-pack/imgs/exception_scenario_TCC.png)    

### (6). Servicecomb Pack学习目录
> ["Servicecomb Pack之Saga+Docker环境搭建(二)"](/2021/05/04/Servicecomb-Saga-Docker.html)   
> ["Servicecomb Pack之Saga本地环境搭建(三)"](/2021/05/04/Servicecomb-Saga.html)  
> ["Servicecomb Pack之微服务下分布式事务案例(四)"](/2021/05/04/Servicecomb-Saga-Booking.html)    
> ["Servicecomb Pack之全局事务@SagaStart(五)"](/2021/05/04/Servicecomb-Saga-SagaStart.html)     
> ["Servicecomb Pack之全局事务是如何传播的(六)"](/2021/05/04/Servicecomb-Saga-Propagation.html)  
> ["Servicecomb Pack之分支事务@Compensable(七)"](/2021/05/04/Servicecomb-Saga-Compensable.html)    
> ["Servicecomb Pack之分支事务@Compensable(八)"](/2021/05/04/Servicecomb-Saga-Compensable-2.html)    

### (7). 总结
> 满怀着期望看ServiceComb Pack源码,但是,失望还是比较大的,我以客观的身份来评价ServiceComb Pack:  
> 1. 框架组件的命名,以阿里的Seata为例,它的命名基本是遵循着规范(XA)来命名的,不会自己创造一些独有的名词,而alpha/omega,我不清楚大部份程序员,能否见名思义?        
> 2. ServiceComb Pack开源社区很薄弱,我遇到的这个Bug(CallbackContext),在2019年就已经有人提出来了,但是,至今未修复(0.5版本的Akka可能修复了).["CallbackContext Bug参考"](https://github.com/apache/servicecomb-pack/issues/590).      
> 3. ServiceComb Pack在任何地方犯错都可以,但是,不应该在主干流程上犯错,同时,也意味着:Saga模式在0.5版本之前,根本就不能用.       
> 4. ServiceComb Pack在0.5版本,增加了Akka来做状态机,看官方文档,需要依赖:ES+Kafka(Redis),整个框架一下子就变得非常的重量级了,意味着:需要维护更多的资源(ES/KAKFA).   
> 5. 所以,暂时放弃:ServiceComb Pack源码探索,因为:要把时间花在值得的位置.     