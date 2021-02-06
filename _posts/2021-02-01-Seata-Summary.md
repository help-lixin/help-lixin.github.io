---
layout: post
title: 'Seata 简介(一)'
date: 2021-02-01
author: 李新
tags: Seata Seata源码
---

### (1). Seata是什么?
> Seata是一款开源的分布式事务解决方案,致力于提供高性能和简单易用的分布式事务服务.<font color='red'>Seata将为用户提供了AT、TCC、SAGA 和 XA 事务模式,为用户打造一站式的分布式解决方案.</font>    

### (2). Seata架构图
> 图片是在百度搜索到的,未能找到相应的博客地址. 

!["Seata框架图"](/assets/seata/imgs/seata-architecture.jpg)

> Transaction Coordinator(TC): 事务协调器,是一个独立的中间件,需要独立部署,它维护着全局事务的运行状态,它主要负责以下工作:  
1. 接受TM发起全局事务的commit与rollback.   
2. 与RM通信协调各分支事务的commit与rollback.   

> Transction Manager(TM) : 事务管理器,TM需要嵌入业务应用程序中工作,它主要负责以下工作:  
1. 负责开启一个全局事务.    
2. 并最终向TC发起全局commit或rollback指令.   

> Resource Manager(RM) : 分支事务参与者,RM也是需要嵌入业务应用程序中工作,它主要负责以下工作:  
1. 分支事务的注册.
2. 状态汇报.   
3. 接受TC(事务协调器)的指令,驱动分支事务的commit或rollback指令.   

### (3). Seata介绍
> ["Seata AT模式分析(二)"](/2021/01/28/Seata-AT.html)   
> ["Seata TCC模式分析(三)"](/2021/01/28/Seata-TCC.html)   
> ["Seata Saga模式分析(四)"](/2021/01/28/Seata-Saga.html)   
> ["Seata 源码下载并编译(五)"](/2021/01/28/Seata-Source-Compile.html)   
> ["Seata AT模式之入门案例(六)"](/2021/01/28/Seata-AT-Example.html)  



### (4). Seata AT源码(全局事务处理)
> ["Seata GlobalTransactionScanner(一)"](/2021/01/29/Seata-Source-GlobalTransactionScanner.html)   
> ["Seata TMClient(二)"](/2021/01/29/Seata-Source-TMClient.html)       
> ["Seata RMClient(三)"](/2021/01/29/Seata-Source-RMClient.html)    
> ["Seata 全局事务处理之GlobalTransactionalInterceptor(四)"](/2021/01/29/Seata-Source-GlobalTransactionalInterceptor.html)    
> ["Seata 全局事务处理之GlobalTransactionalInterceptor(五)"](/2021/01/29/Seata-Source-TransactionalTemplate.html)    
> ["Seata 全局事务处理之TransactionalTemplate(六)"](/2021/01/29/Seata-Source-TransactionalTemplate.html)    
> ["Seata 全局事务处理之GlobalTransaction(七)"](/2021/01/29/Seata-Source-GlobalTransaction.html)    
> ["Seata 全局事务处理之TransactionManager(八)"](/2021/01/29/Seata-Source-TransactionManager.html)

### (5). Seata AT源码(分支事务处理)
> ["Seata 分支事务处理之DataSourceProxy初始化(一)"](/2021/01/29/Seata-Source-DataSourceProxy-new.html)    
> ["Seata 分支事务处理之ResourceManager(二)"](/2021/01/29/Seata-Source-ResourceManager.html)    
> ["Seata 分支事务处理之DataSourceProxy获取连接(三)"](/2021/01/29/Seata-Source-DataSourceProxy-getConnection.html)    
> ["Seata 分支事务处理之UpdateExecutor(四)"](/2021/01/29/Seata-Source-UpdateExecutor.html)    
> ["Seata 分支事务处理之ConnectionProxy提交/回滚事务详解(五)"](/2021/01/29/Seata-Source-ConnectionProxy-commit.html)    
> ["Seata 分支事务处理之RmBranchCommitProcessor(六)"](/2021/01/29/Seata-Source-RmBranchCommitProcessor.html)    
> ["Seata 分支事务处理之RmBranchRollbackProcessor(七)"](/2021/01/29/Seata-Source-RmBranchRollbackProcessor.html)    


### (6). Seata TCC源码(全局事务处理)
> ["Seata  TCC 全局事务之GlobalTransactionScanner(一)"](/2021/01/29/Seata-Source-TCC-GlobalTransactionScanner.html)    
> ["Seata  TCC全局事务之TCCBeanParserUtils(二)"](/2021/01/29/Seata-Source-TCC-TCCBeanParserUtils.html)    
> 余下TCC全局事务内容与AT事务内容是一样的,主要职责如下:    
> 1. 针对方法上拥有@GlobalTransactional和@GlobalTransactional的注解进行代码增强.  
> 2. 向TC注册全局事务begin(DefaultTransactionManager.begin).   
> 3. 执行业务代码.  
> 4. 通知TC,全局事务commit或rollback(DefaultTransactionManager.commit/rollback).   

### (6). Seata TCC源码(分支事务处理)
> ["Seata  TCC分支事务之TccActionInterceptor(一)"](/2021/01/29/Seata-Source-TCC-TccActionInterceptor.html)    
> ["Seata  TCC分支事务之ActionInterceptorHandler(二)"](/2021/01/29/Seata-Source-TCC-ActionInterceptorHandler.html)      


