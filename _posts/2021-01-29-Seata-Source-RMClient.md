---
layout: post
title: 'Seata  RMClient(三)'
date: 2021-01-29
author: 李新
tags: Seata源码
---

### (1). 先看下RMClient类的结构图
!["RMClient类结构图"](/assets/seata/imgs/seata-RMClient.jpg)

> RM全称为:Resource Mangaer,它主要与TC进行通信,它的职责是:  
> 1. 分支事务的注册和汇报
> 2. 接受TC(事务协调器)的指令,驱动分支事务的commit或rollback指令.   
> 3. 从类的结构图上看:RmNettyRemotingClient相比TmNettyRemotingClient增加了不少的功能.   

### (2). RMClient.init
```
public static void init(String applicationId, String transactionServiceGroup) {
	// 构建:RmNettyRemotingClient(这里和TmNettyRemotingClient都差不多,我就不分析了)
	RmNettyRemotingClient rmNettyRemotingClient = RmNettyRemotingClient.getInstance(applicationId, transactionServiceGroup);
	
	// ***********************************************************
	// 创建了ResourceManager的实现类(DefaultResourceManager)
	// 从ResourceManager接口签名就知道:它主要负责创建分支事务/汇报/锁undolog...
	// ***********************************************************
	rmNettyRemotingClient.setResourceManager(DefaultResourceManager.get());
	
	// ***********************************************************
	// 创建了AbstractRMHandler的实现类(DefaultRMHandler)
	// 从AbstractRMHandler接口签名就知道:它主要负责处理Netty的请求.
	// ***********************************************************
	rmNettyRemotingClient.setTransactionMessageHandler(DefaultRMHandler.get());
	rmNettyRemotingClient.init();
```
### (3). RmNettyRemotingClient.registerProcessor
> 我们看下RM注册了哪些RemotingProcessor

```
// RmNettyRemotingClient注册的:RemotingProcessor明显要比:TmNettyRemotingClient要多很多.
private void registerProcessor() {
	
	// 分支事务的commit处理
	// 1.registry rm client handle branch commit processor
	RmBranchCommitProcessor rmBranchCommitProcessor = new RmBranchCommitProcessor(getTransactionMessageHandler(), this);
	super.registerProcessor(MessageType.TYPE_BRANCH_COMMIT, rmBranchCommitProcessor, messageExecutor);
	
	// 分支事务的rollback处理
	// 2.registry rm client handle branch commit processor
	RmBranchRollbackProcessor rmBranchRollbackProcessor = new RmBranchRollbackProcessor(getTransactionMessageHandler(), this);
	super.registerProcessor(MessageType.TYPE_BRANCH_ROLLBACK, rmBranchRollbackProcessor, messageExecutor);
	
	// undo log处理
	// 3.registry rm handler undo log processor
	RmUndoLogProcessor rmUndoLogProcessor = new RmUndoLogProcessor(getTransactionMessageHandler());
	super.registerProcessor(MessageType.TYPE_RM_DELETE_UNDOLOG, rmUndoLogProcessor, messageExecutor);
	
	// 向tc进行汇报处理
	// 4.registry TC response processor
	ClientOnResponseProcessor onResponseProcessor =
		new ClientOnResponseProcessor(mergeMsgMap, super.getFutures(), getTransactionMessageHandler());
	
	super.registerProcessor(MessageType.TYPE_SEATA_MERGE_RESULT, onResponseProcessor, null);
	super.registerProcessor(MessageType.TYPE_BRANCH_REGISTER_RESULT, onResponseProcessor, null);
	super.registerProcessor(MessageType.TYPE_BRANCH_STATUS_REPORT_RESULT, onResponseProcessor, null);
	super.registerProcessor(MessageType.TYPE_GLOBAL_LOCK_QUERY_RESULT, onResponseProcessor, null);
	super.registerProcessor(MessageType.TYPE_REG_RM_RESULT, onResponseProcessor, null);
	
	// 心跳
	// 5.registry heartbeat message processor
	ClientHeartbeatProcessor clientHeartbeatProcessor = new ClientHeartbeatProcessor();
	super.registerProcessor(MessageType.TYPE_HEARTBEAT_MSG, clientHeartbeatProcessor, null);
}
```

### (4). 总结
> RMClient相比TMClient的业务要多很多.从类的结构图上就能看得出来.  
> 我这里把:RMClient独特的业务点给摘抄出来了,其它(比如:Netty如何处理消息)和上一节内容是一样的.就不再重复抄代码了.  
> 有一些接口的功能,在这里不分析,等到真正用到的时候,我们再跟踪进来分析下.
