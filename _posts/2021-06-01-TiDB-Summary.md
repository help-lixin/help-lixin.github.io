---
layout: post
title: 'TiDB 介绍(一)'
date: 2021-06-01
author: 李新
tags:  TiDB
---

### (1). TiDB介绍
> [""TiDB"](https://docs.pingcap.com/zh/tidb/stable)是PingCAP公司自主设计、研发的开源分布式关系型数据库,是一款同时支持在线事务处理与在线分析处理(Hybrid Transactional and Analytical Processing, HTAP)的融合型分布式数据库产品.  
> 具备水平扩容或者缩容、金融级高可用、实时 HTAP、云原生的分布式数据库、兼容 MySQL 5.7 协议和 MySQL 生态等重要特性.目标是为用户提供一站式OLTP(Online Transactional Processing)、OLAP(Online Analytical Processing)、HTAP 解决方案.
> TiDB 适合高可用、强一致要求较高、数据规模较大等各种应用场景.     

### (2). TiDB学习目录
> ["TiDB 架构详解(二)"](/2021/06/01/TiDB-Architecture.html)    
> ["TiDB 单机安装(三)"](/2021/06/01/TiDB-Simple-Install.html)   
> ["TiDB Docker集群安装(四)"](/2021/06/01/TiDB-Docker-Cluster-Install.html)   

### (3). 总结
> 优点就不用说了,TiDB在分布式的情况下,有着一整套完善的生态圈来解决问题.   
> 个人觉得缺点就是:成本问题(上生产最低要求:16核32G * 5台 / 4核8G * 2台 ),数据是随着时间增长而增长的,成本也应该是随着时间增长而增长的. 