---
layout: post
title: 'SOFAJRaft总结' 
date: 2023-01-20
author: 李新
tags:  SOFAJRaft
---


### (1). 前因
> 对最新的Camunda 8(Zeebe)非常的好奇,为什么呢?因为传统的工作流跟不上微服务的脚步,我列举一个案例:     
> 本人曾在宇龙计算机科技工作过,公司按照业务流进行了拆分(比如:HR/OA/PLM...),企业要求通过单点登录之后,在待审批列表,能看到所有业务系统的待审批任务,而不是跳到每个业务系统去操作.     
> 解决方案有两种:    
> 1. 待审批列表,通过Ajax去各个业务系统加载待审批数据进行渲染,缺点:分页肯定不准确的啦,优点:实现起来,相当的简单.     
> 2. 对Activit进行扩展,有幸看过Activit源码,底层每一次操作时,都会发布成事件,订阅这些事件,把待办任务写到一张表里,缺点:在互联网时代,该表的数据量相当的大,而且,所有业务系统要连接到这个数据源.优点:把分散的东西,聚合在一起.      

> 总结,上面两种方案,实际上都是很拐扭,各业务系统都拥有JPBM的一套表,仅仅,是为了解决:待审批列表问题展示,我们要深思,能否"各业务系统"只提供"工作流组件的钩子实现",而,工作流引擎来驱动着工作流向前走?这时候Zeebe就出现了,它也是未来Camunda 8的驱动引擎,
> 在看Zeebe的源码时,发现它直接clone了一份Atomix的源码,而且,是在Atomix的基础上进行了扩展,没办法,要Hold住Zeebe是必然要Hold住Atomix,不得不深入去学习,否则,纯粹的使用工具,或拿着RAFT协议,夸夸其谈(纸上谈兵),没有意义,重要的是:自己将来可能会这上面做扩展或改造.  

### (2). 经历
> 近半年来,除了考驾照,帮父母照料生意之外,抽出一些时间来学习RAFT,学习过程颇为陡峭,至少想放弃过100多次,不是简简单的理解RAFT协议那么简单来着的,很多时候,要考虑别人的框架为什么要那样写,致使自己陷入太深. 
> RAFT源码学习过程为:XRaft/Atomix/JRaft,对于这几个框架的源码,有一些没有写文档(Atomix后面会慢慢补上),但是,对于JRaft写了一份文档,因为,JRaft的架构图,让我一眼就穿内部组件,方便学习.  

### (3). JRaft框架图
> 通过对JRaft的源码剖析,JRaft在开发之前,是肯定做了很详细的设计来着的,官网虽然简简单单几字,但是,对于我来说:总比没有的架构图参考比较好.  

!["JRaft框架图"](/assets/jraft/imgs/jraft-engine.png)
### (4). JRaft用到了哪些设计模式
> 1. 工厂模式(RaftRpcServerFactory). 
> 2. 单例模式(NodeManager).
> 3. 中介者模式(RpcServer注册业务处理).  
> 4. 门面模式(NodeImpl聚合了所有的类). 

### (5). JRaft源码剖析目录

> ["SOFAJRaft源码之RaftRpcServerFactory(一)"](/2022/12/28/SOFAJRaft-RaftRpcServerFactory.html)  
> ["SOFAJRaft源码之RocksDBLogStorage初始化(二)"](/2022/12/28/SOFAJRaft-RocksDBLogStorage-Init.html)  
> ["SOFAJRaft源码之RocksDBLogStorage常用方法剖析(三)"](/2022/12/28/SOFAJRaft-RocksDBLogStorage.html)  
> ["SOFAJRaft源码之LogManager初始化(四)"](/2022/12/28/SOFAJRaft-LogManager-Init.html)  
> ["SOFAJRaft源码之LogManager常用方法剖析(五)"](/2022/12/28/SOFAJRaft-LogManager.html)  
> ["SOFAJRaft源码之RaftMetaStorage(六)"](/2022/12/28/SOFAJRaft-RaftMetaStorage.html)  
> ["SOFAJRaft源码之SnapshotWriter(七)"](/2022/12/28/SOFAJRaft-SnapshotWriter.html)  
> ["SOFAJRaft源码之SnapshotReader(八)"](/2022/12/28/SOFAJRaft-SnapshotReader.html)  
> ["SOFAJRaft源码之SnapshotStorage(九)"](/2022/12/28/SOFAJRaft-SnapshotStorage.html)  
> ["SOFAJRaft源码之FSMCaller上篇(十)"](/2022/12/28/SOFAJRaft-FSMCaller-1.html)   
> ["SOFAJRaft源码之FSMCaller中篇(十一)"](/2022/12/28/SOFAJRaft-FSMCaller-2.html)   
> ["SOFAJRaft源码之FSMCaller下篇(十二)"](/2022/12/28/SOFAJRaft-FSMCaller-3.html)  
> ["SOFAJRaft源码之ClosureQueue(十三)"](/2022/12/28/SOFAJRaft-ClosureQueue.html)  
> ["SOFAJRaft源码之IteratorImpl(十四)"](/2022/12/28/SOFAJRaft-IteratorImpl.html)   
> ["SOFAJRaft源码之StateMachine(十五)"](/2022/12/28/SOFAJRaft-StateMachine.html)  
> ["SOFAJRaft源码之BallotBox之追加任务(十六)"](/2022/12/28/SOFAJRaft-BallotBox-appendPendingTask.html)  
> ["SOFAJRaft源码之BallotBox之日志确认(十七)"](/2022/12/28/SOFAJRaft-BallotBox-commitAt.html)  
> ["SOFAJRaft源码之ClientService初始化(十八)"](/2022/12/28/SOFAJRaft-ClientService.html)   
> ["SOFAJRaft源码之RpcServer初始化(十九)"](/2022/12/28/SOFAJRaft-RpcServer.html)  
> ["SOFAJRaft源码之HashedWheelTimer(二十)"](/2022/12/28/SOFAJRaft-HashedWheelTimer.html)  
> [SOFAJRaft源码之RepeatedTimer(二十一)](/2022/12/28/SOFAJRaft-RepeatedTimer.html)  
> ["SOFAJRaft源码之RaftGroupService(二十二)"](/2023/01/11/SOFAJRaft-RaftGroupService.html) 
> ["SOFAJRaft源码之Node初始化(二十三)"](/2023/01/11/SOFAJRaft-Node-Init.html)  
> ["SOFAJRaft源码之Node发起预投票请求(二十四)"](/2023/01/11/SOFAJRaft-Node-PreVote.html)  
> ["SOFAJRaft源码之Node预投票请求处理(二十五)"](/2023/01/11/SOFAJRaft-Node-PreVote-Handler.html)   
> ["SOFAJRaft源码之Node发起投票请求(二十六)"](/2023/01/11/SOFAJRaft-Node-Vote.html)  
> ["SOFAJRaft源码之Node投票请求处理(二十七)"](/2023/01/11/SOFAJRaft-Node-Vote-Handler.html)   
> ["SOFAJRaft源码之Node提交任务(二十八)"](/2023/01/11/SOFAJRaft-Node-Apply-Task.html)  

### (6). 总结
阿里开源的框架,代码组件,与框架图基本一一呼应,虽然,只是支言片语,但是,有这一张图,足已让我轻松捍动它的源码了,感谢阿里开源知识分享.    