---
layout: post
title: 'Nacos 基本概念(一)'
date: 2019-05-01
author: 李新
tags: Nacos
---

### (1). Nacos管理模型
> 对于Nacos的配置管理,可以通过:Namespace、group、DataID 来定位一个配置集(配置的集合). 
> 下面这幅图,就是(Nacos data model)数据模型:

!["Nacos架构模型"](/assets/nacos/imgs/nacos-data-model.jpeg)

### (2). Namespace
> Namespace可用于进行不同环境的配置隔离(例如:开发,测试,生产)
### (3). Group(配置分组)
> Group是对DataId(配置集)进行分组.  
> Group常见场景,用于区分:<font color='red'>不同的系统名称,项目名称,应用名称.</font>    
### (4). Service/DataId
> DataId代表:一个配置集,所谓的配置集,就是:<font color='red'>一个配置文件包含的配置集合.</font>   
### (5). 最佳实践
> Namespace  : 环境,比如:45a6112b-a866-4e92-a8d6-9e440bcdebbe   
> Group      : 项目名称(微服务名称),比如:erp:sso-service     
> DataId     : 具体的配置文件,比如:application.properties,datasource.properties      