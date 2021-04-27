---
layout: post
title: 'Hadoop MapReduce(五)'
date: 2018-03-24
author: 李新
tags: Hadoop
---

### (1). MapReduce介绍

> MapReduce是一种编程模型,它能够把应用程序分割成许多的小的工作单元,并把这些单元放到任何集群节点上执行.

### (2). MapReduce架构

!["Hadoop MapReduce架构"](/assets/hadoop/imgs/hadoop-mapreduce-architecture.png )

### (3). ResourceManager

> 接受用户的计算请求任务,并负责集群的资源管理

### (4). NodeManger

> 负责执行主节点AppMaster分配的任务. 
