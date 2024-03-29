---
layout: post
title: 'CAS总结' 
date: 2021-10-26
author: 李新
tags:  CAS
---

### (1). 概述
为什么要看CAS源码,以前是一家跨境电商公司,针对单点登录(SSO)这一块,是自己实现了SSO的协议来做的(Passport),当初,朋友说:因为CAS代码太复杂了,所以,才会自研了一套,这家公司有很多技术都是基于自研.
原因:最高层会根据外界的需求,要求技术要能做到随时调整技术,而,对于技术若是偏于简单使用,将无法适应业务,再加上,架构师就是一块夹心饼,为了能够适用业务的变化,就要求对源码要有80%的了解,当你无法撑控全局时就会退而其次选择自研.    
大部份Java源码看得差不多了,最近有一点点时间,就想把SSO的源码稍微看一下.   

### (2). 注意事项
在看Cas源码之前,还是建议要学习下:Spring Web Flow,因为,Cas底层是基于Spring Web Flow进行开发的.

### (3). CAS Filter学习目录
+ ["CAS基本概念介绍(一)"](/2021/10/26/Cas-Concept.html)     
+ ["CAS脚手架源码之编译并运行(二)"](/2021/10/26/Cas-Source-Compile-Run.html)   
+ ["CAS源码下载以及工程目录介绍(三)"](/2021/10/26/Cas-Source-Concept.html)   
+ ["CAS分析源码之切入点在哪(四)"](/2021/10/26/Cas-Source-Main.html)   
+ ["CAS源码之ClientInfoThreadLocalFilter(四)"](/2021/10/26/Cas-Source-ClientInfoThreadLocalFilter.html)    
+ ["CAS源码之ThreadContextMDCServletFilter(五)"](/2021/10/26/Cas-Source-ThreadContextMDCServletFilter.html)    
+ ["CAS源码之DelegatingFilterProxyRegistrationBean(六)"](/2021/10/26/Cas-Source-DelegatingFilterProxyRegistrationBean.html)   
+ ["CAS源码之向Spring MVC中手工注册Controller(七)"](/2021/10/26/Cas-Source-Register-Controller.html)   

### (4). 