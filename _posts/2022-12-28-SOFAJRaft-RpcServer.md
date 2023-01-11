---
layout: post
title: 'SOFAJRaft源码之RpcServer初始化(十九)' 
date: 2022-12-28
author: 李新
tags:  SOFAJRaft
---

### (1). 概述
在准备剖析Replicator时,发现它会与网络通信有一些纠缠不清,因为与网络相关,想要剖析整个链路,不想再进行Mock了,所以,这一小篇主要剖析与网络相关的类:RpcServer

### (2). RpcServer UML图解
!["RpcServer UML图"](/assets/jraft/imgs/RpcServer-ClassDiagram.jpg)

### (3). RpcServer接口
> 通过RpcServer接口就能猜出来,这个接口主要用于注册RPC请求处理的.  

```
public interface RpcServer extends Lifecycle<Void> {

    void registerConnectionClosedEventListener(final ConnectionClosedEventListener listener);

    void registerProcessor(final RpcProcessor<?> processor);

    int boundPort();
}
```
### (4). RpcProcessor接口
```
public interface RpcProcessor<T> {

    // *****************************************************************
	// 处理RPC请求
	// *****************************************************************
    void handleRequest(final RpcContext rpcCtx, final T request);

    String interest();

    default Executor executor() {
        return null;
    }

    default ExecutorSelector executorSelector() {
        return null;
    }
	
    interface ExecutorSelector {
        Executor select(final String reqClass, final Object reqHeader);
    }
}
```
### (5). RpcServer初始化案例分析
> 通过RaftRpcServerFactory创建:RpcServer,并注册:Processor.  


```
// 创建RPC请求处理器.
// here use same RPC server for raft and business. It also can be seperated generally
final RpcServer rpcServer = RaftRpcServerFactory.createRaftRpcServer(serverId.getEndpoint());

// ***************************************************************************
// 业务处理的最终实现类
// register business processor
// CounterService是业务方法,在业务方法里,又Hold住了this,这个this里包含着(Node/StateMachine)
// 我的大概理解是:RpcServer --> Processor(GetValueRequestProcessor/IncrementAndGetRequestProcessor) --> CounterService --> Node
// ***************************************************************************
CounterService counterService = new CounterServiceImpl(this);

// ***************************************************************************
// 为RpcServer配置:Processor
// Processor: 理解为对协议进行解码与处理即可.
// ***************************************************************************
// 获取请求处理,并委托给:CounterService
rpcServer.registerProcessor(new GetValueRequestProcessor(counterService));
// 安全自增请求处理,并委托给:CounterService
rpcServer.registerProcessor(new IncrementAndGetRequestProcessor(counterService));
```
### (6). 总结
从UML上就能看出来,RpcServer的职责,它在Netty的基础上创建进行了封装,并提供API,注册RPC请求处理器(RpcProcessor),在这里不细去分析具体的RpcProcessor,留到后面当发起,具体请求时,我们拿一条链路来分析. 