---
layout: post
title: 'Hadoop HDFS'
date: 2018-03-24
author: 李新
tags: Hadoop
---

### (1). HDFS介绍

> HDFS,是Hadoop Distributed File System的简称,是Hadoop抽象文件系统的一种实现.HDFS主要用于**存储非常大的文件**.

### (2). HDFS架构

!["Hadoop HDFS架构图"](/assets/hadoop/imgs/hadoop-hdfs-architecture.png)

### (3). NameNode

> NameNode集群中的**元数据节点**,管理元数据(文件大小/文件位置/文件权限),主要用于管理集群中的各种元数据.

### (4). SecondaryNameNode

> SencodaryNameNode主要用于Hadoop当中**元数据信息的辅助管理.**

### (5). DataNode

> 集群中的数据存储节点,主要用于**存储数据**
