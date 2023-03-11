---
layout: post
title: 'Camundate 核心表结构了解' 
date: 2022-02-08
author: 李新
tags:  Camunda
---

### (1). 概述
最近在了解工作流,为什么选择Camunda,是因为Camunda 8面向了云原生,吞吐量有所提升来着的,这几天学习Camunda 7时,发现,Camunda 7的设计还是可以的,曾经我提到Activiti的问题,在Camunda都有做解决来着的. 

### (2). 流程部署相关的表
```
ACT_GE_BYTEARRAY               :  存储流程部署的xml信息
ACT_RE_DEPLOYMENT              :  一次流程部署
ACT_RE_PROCDEF                 :  存储部署时的相关信息(key)
```
### (3). 流程执行相关的表
```
ACT_RU_EXECUTION               :  流程执行时,当前运行节点信息(比如:人工审批)
ACT_RU_TASK                    :  流程任务(比如:张三/李四审批),会在这张表里记录这些信息
ACT_RU_EXT_TASK                : ServiceTask定义的外部任务
```
### (4). 流程历史表
```
ACT_HI_PROCINST                : 流程实例信息
ACT_HI_ACTINST                 : 流程实例执行过的所有信息.

```
### (5). Camunda优化
我记忆中,有一张表是流程实例表,现在好像去掉这张表了,流程一开始运行,就直接把实例信息写到了:ACT_HI_PROCINST表里.  