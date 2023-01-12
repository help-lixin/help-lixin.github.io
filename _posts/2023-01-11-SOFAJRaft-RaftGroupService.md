---
layout: post
title: 'SOFAJRaft源码之RaftGroupService(二十二)' 
date: 2023-01-11
author: 李新
tags:  SOFAJRaft
---

### (1). 概述
本来是要分析Node的,而且,自己对Node的理解是网络通信的Server端,结果,打脸,发现:它(Node)居然是网络通信的Client,所以,这一篇主要分析与网络通信Server层相关的内容,即:RaftGroupService. 

### (2). RaftGroupService静态代码
```
static {
	// 通过ProtoBuf工厂,加载静态代码,在这里不进行详解了,不是,我要去深入的重点.
	ProtobufMsgFactory.load();
}
```
### (3). RaftGroupService初始化
```
public RaftGroupService(
              // RAFT分组的ID
             final String groupId, 
			 // 当前节点
			 final PeerId serverId, 
			 // 节点配置
			 final NodeOptions nodeOptions,
			 // 外部传入:RpcServer
			 final RpcServer rpcServer, 
			 // 是否共享rpc server
			 final boolean sharedRpcServer) {
	super();
	this.groupId = groupId;
	this.serverId = serverId;
	this.nodeOptions = nodeOptions;
	this.rpcServer = rpcServer;
	this.sharedRpcServer = sharedRpcServer;
}
```
### (4). RaftGroupService.start
```
public synchronized Node start(final boolean startRpcServer) {
	if (this.started) {
		return this.node;
	}
	if (this.serverId == null || this.serverId.getEndpoint() == null
		|| this.serverId.getEndpoint().equals(new Endpoint(Utils.IP_ANY, 0))) {
		throw new IllegalArgumentException("Blank serverId:" + this.serverId);
	}
	if (StringUtils.isBlank(this.groupId)) {
		throw new IllegalArgumentException("Blank group id:" + this.groupId);
	}

	//Adds RPC server to Server.
	NodeManager.getInstance().addAddress(this.serverId.getEndpoint());
	
	
	// ***********************************************************************************
	// 典型的工厂模工创建:Node,在这里先不关心是如何创建的,留到下一篇去分析. 
	// ***********************************************************************************
	this.node = RaftServiceFactory.createAndInitRaftNode(this.groupId, this.serverId, this.nodeOptions);
	
	if (startRpcServer) {
		this.rpcServer.init(null);
	} else {
		LOG.warn("RPC server is not started in RaftGroupService.");
	}
	
	this.started = true;
	LOG.info("Start the RaftGroupService successfully.");
	return this.node;
} // end 
```
### (5). 总结
> 所以,RaftGroupService的目的是:   
> 1. 通过RaftServiceFactory创建:Node.    
> 2. Hold住RpcServer,并进行初始化.  