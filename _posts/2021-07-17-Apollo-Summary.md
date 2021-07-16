---
layout: post
title: 'Apollo 总结' 
date: 2021-07-17
author: 李新
tags:  Apollo
---

### (1). 概述
花了一天半的时间,稍微的学习下Apollo,现对其,进行一个总结:   
1) 总体来说Apollo的架构设计得比较复杂,一般人不看官网说明,看起来就会有点困难.     
2) 类名称/方法名称,基本上是固名思义,而且,也用到了大量的设计模式(单例/组合/观察者).      
3) 属性命名这一块,我反正有点不适用.     
4) 自身需要的配置化的信息,都将会转换到System.setProperty里,可是,你若是像Tomcat多应用的情况下,不知道官网是怎么解决的.  

### (2). 为什么研究Apollo
1) Eureka属于CAP中的AP模式,可以想像成蝗虫一样,快速的传播,而,Nacos用到了DB,有DB就会有限制(当然,也可以只用Nacos中的配置中心).   
2) Nacos在UI上的权限控制很粗糙,总感觉就是个半成品.    
3) Apollo就专心做一件事.   

### (3). Apollo学习目录
+ ["Apollo架构深入浅出(一)"](/2021/07/16/Apollo-Architecture.html)   
+ ["Apollo安装与部署(二)"](/2021/07/16/Apollo-Deploy.html)  
+ ["Apollo简单使用与集成(三)"](/2021/07/16/Apollo-JavaClient-Integration.html)  
+ ["Apollo源码学习之:配置热更新原理(四)"](/2021/07/16/Apollo-AutoUpdateConfigChangeListener.html)
+ ["Apollo源码学习之:@EnableApolloConfig作用(五)"](/2021/07/16/Apollo-EnableApolloConfig.html)  
+ ["Apollo源码学习之:Spring无缝整合(六)"](/2021/07/16/Apollo-ApolloApplicationContextInitializer.html)  
+ ["Apollo源码学习之:拉取配置过程部析(七)"](/2021/07/16/Apollo-RemoteConfigRepository.html)   

