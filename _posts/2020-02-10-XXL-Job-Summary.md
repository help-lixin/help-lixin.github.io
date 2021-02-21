---
layout: post
title: 'XXL-Job 简介(一)'
date: 2020-02-11
author: 李新
tags: XXL-Job源码
---

### (1). XXL-Job是什么?
> XXL-JOB是一个轻量级分布式任务调度框架,其核心设计目标是开发迅速、学习简单、轻量级、易扩展.  

### (2). 定时任务技术选型
> Quartz缺点:定时任务的配置信息和执行逻辑过度耦合在一起.   
> Elastic-job缺点:过渡依赖于ZK了,如果定时任务是跨机房呢?   
> XXL-Job缺点:定时任务不支持时区,但,总体来说没有Elastic-Job那么重(依赖ZK).又提供了Quartz没有的功能.   

### (3). XXL-Job架构
!["XXL-Job架构图"](/assets/xxl-job/imgs/xxl-job.png)

> xxl-job-admin : UI界面,负责对:执行器管理/任务管理/用户管理...
> app : 应用程序,接受:xxl-job-admin分配的任务.  

### (4). XXL-Job源码目录
> ["XXL-Job XxlJobConfig(二)"](/2021/02/21/XXL-Job-XxlJobConfig.html)    
> ["XXL-Job XxlJobSpringExecutor(三)"](/2021/02/21/XXL-Job-XxlJobSpringExecutor.html)    