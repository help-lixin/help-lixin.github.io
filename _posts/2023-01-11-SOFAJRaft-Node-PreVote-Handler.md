---
layout: post
title: 'SOFAJRaft源码之Node预投票请求处理(二十五)' 
date: 2023-01-11
author: 李新
tags:  SOFAJRaft
---

### (1). 概述
上一篇,分析的是Node发起预投票请求,这一篇主要分析,Node发起预投票请求之后,服务端处理流程. 

### (2). 先看下注册网络请求部份的代码(RaftRpcServerFactory.addRaftRequestProcessors)
> 为什么这里要看下这段代码,而不是在剖析RaftRpcServerFactory时看,因为,在剖析RaftRpcServerFactory时,不要太过于关注细节,在真正遇到阻碍时,才回过来细看一下这段代码.  

```
public static void addRaftRequestProcessors(final RpcServer rpcServer, final Executor raftExecutor,
                                                final Executor cliExecutor) {
	// **************************************************************************
	// Netty接受投票请求和心跳请求的处理
	// **************************************************************************
	rpcServer.registerProcessor(new RequestVoteRequestProcessor(raftExecutor));
	rpcServer.registerProcessor(new PingRequestProcessor());
}
```
### (3). BoltRpcServer.registerProcessor
```
public void registerProcessor(final RpcProcessor processor) {
	// *********************************************************************
	// 1. 创建:AsyncUserProcessor对RpcProcessor(RequestVoteRequestProcessor)进行包裹
	// *********************************************************************
	this.rpcServer.registerUserProcessor(new AsyncUserProcessor<Object>() {

		@SuppressWarnings("unchecked")
		@Override
		public void handleRequest(final BizContext bizCtx, final AsyncContext asyncCtx, final Object request) {
			
			// *********************************************************************
			// 2. 创建RpcContext
			// *********************************************************************
			final RpcContext rpcCtx = new RpcContext() {

				@Override
				public void sendResponse(final Object responseObj) {
					asyncCtx.sendResponse(responseObj);
				}

				@Override
				public Connection getConnection() {
					com.alipay.remoting.Connection conn = bizCtx.getConnection();
					if (conn == null) {
						return null;
					}
					return new BoltConnection(conn);
				}

				@Override
				public String getRemoteAddress() {
					return bizCtx.getRemoteAddress();
				}
			};
            
			// ********************************************************************
			// 3. 最后才是移交给业务处理(RequestVoteRequestProcessor)
			// ********************************************************************
			processor.handleRequest(rpcCtx, request);
		} // end 
	}); // end registerUserProcessor
}
```
### (4). RequestVoteRequestProcessor继承关系
> 前面对RpcProcessor进行了剖析,里面最重要的方法是:handleRequest. 

```
com.alipay.sofa.jraft.rpc.RpcProcessor
	com.alipay.sofa.jraft.rpc.RpcRequestProcessor
		com.alipay.sofa.jraft.rpc.impl.core.NodeRequestProcessor
			com.alipay.sofa.jraft.rpc.impl.core.RequestVoteRequestProcessor
```
### (5). RpcRequestProcessor.handleRequest
> 上面剖析了,当注册RpcProcessor时,会被AsyncUserProcessor进行包裹,创建:RpcContext,最终才会回调到业务handleRequest上. 

```
public void handleRequest(final RpcContext rpcCtx, final T request) {
	try {
		// ********************************************************************************************
		// RpcRequestProcessor是个抽象类,最终委派给了子类(NodeRequestProcessor)
		// ********************************************************************************************
		final Message msg = processRequest(request, new RpcRequestClosure(rpcCtx, this.defaultResp));
		if (msg != null) {
			rpcCtx.sendResponse(msg);
		}
	} catch (final Throwable t) {
		LOG.error("handleRequest {} failed", request, t);
		rpcCtx.sendResponse(RpcFactoryHelper //
			.responseFactory() //
			.newResponse(defaultResp(), -1, "handleRequest internal error"));
	}
} // end handleRequest


// ********************************************************************************************
// 定义抽象方法,给子类去实现
// ********************************************************************************************
public abstract Message processRequest(final T request, final RpcRequestClosure done);
```
### (6). NodeRequestProcessor.processRequest
```
public Message processRequest(final T request, final RpcRequestClosure done) {
	final PeerId peer = new PeerId();
	final String peerIdStr = getPeerId(request);
	if (peer.parse(peerIdStr)) {
		final String groupId = getGroupId(request);
		// *****************************************************************
		// NodeManager类是没有分析过的,不过先不管了,肯定是Node初始化后,往里塞,然后,用的时候再取出来.
		// 典型的单例模式
		// 这里的peer为(127.0.0.1:8082,127.0.0.1:8083)
		// *****************************************************************
		// 根据groupId和peer取出Node对象
		final Node node = NodeManager.getInstance().get(groupId, peer);
		if (node != null) {
			// *****************************************************************
			// 委托给子类处理(RequestVoteRequestProcessor),把Node强制成:RaftServerService
			// *****************************************************************
			return processRequest0((RaftServerService) node, request, done);
		} else {
			return RpcFactoryHelper //
				.responseFactory() //
				.newResponse(defaultResp(), RaftError.ENOENT, "Peer id not found: %s, group: %s", peerIdStr,
					groupId);
		}
	} else {
		return RpcFactoryHelper //
			.responseFactory() //
			.newResponse(defaultResp(), RaftError.EINVAL, "Fail to parse peerId: %s", peerIdStr);
	}
} // end processRequest

// 定义抽象类给子类实现
protected abstract Message processRequest0(final RaftServerService serviceService, final T request, final RpcRequestClosure done);
```
### (7). RequestVoteRequestProcessor.processRequest0
```
// 此处的:RaftServerService就是Node
public Message processRequest0(final RaftServerService service, final RequestVoteRequest request,
                                   final RpcRequestClosure done) {
	if (request.getPreVote()) { // 是否为预投票请求
	    // NodeImpl实现了:RaftServerService
		return service.handlePreVoteRequest(request);
	} else {
		return service.handleRequestVoteRequest(request);
	}
} // end 
```
### (8). Node.handlePreVoteRequest
```
public Message handlePreVoteRequest(final RequestVoteRequest request) {
	boolean doUnlock = true;
	this.writeLock.lock();
	try {
		// 验证本节点(比如:127.0.0.1:8083),的状态是否为活动状态
		if (!this.state.isActive()) {
			LOG.warn("Node {} is not in active state, currTerm={}.", getNodeId(), this.currTerm);
			return RpcFactoryHelper //
				.responseFactory() //
				.newResponse(RequestVoteResponse.getDefaultInstance(), RaftError.EINVAL,
					"Node %s is not in active state, state %s.", getNodeId(), this.state.name());
		}
		
		final PeerId candidateId = new PeerId();
		
		
		// request.getServerId() = 127.0.0.1:8081
		// 看一下请求的:serverid是否正常,并解析到candiateId对象里
		if (!candidateId.parse(request.getServerId())) {
			LOG.warn("Node {} received PreVoteRequest from {} serverId bad format.", getNodeId(),
				request.getServerId());
			return RpcFactoryHelper //
				.responseFactory() //
				.newResponse(RequestVoteResponse.getDefaultInstance(), RaftError.EINVAL,
					"Parse candidateId failed: %s.", request.getServerId());
		}

		// 是否授权通过.
		boolean granted = false;
		
		// 不太理解为什么要用do { } while(false)
		// noinspection ConstantConditions
		do {
			// 验证下节点是否在启动配置里存在,额,手动添加的Peer节点应该是不会走这个逻辑吧! 
			if (!this.conf.contains(candidateId)) {
				LOG.warn("Node {} ignore PreVoteRequest from {} as it is not in conf <{}>.", getNodeId(),
					request.getServerId(), this.conf);
				break;
			}
			
			// 当前节点(127.0.0.1:8083)的leaderId属性,如果存在有值,代表已经有Leader了,不能投票
			if (this.leaderId != null && !this.leaderId.isEmpty() && isCurrentLeaderValid()) {
				LOG.info(
					"Node {} ignore PreVoteRequest from {}, term={}, currTerm={}, because the leader {}'s lease is still valid.",
					getNodeId(), request.getServerId(), request.getTerm(), this.currTerm, this.leaderId);
				break;
			}

			// 验证远程节点(127.0.0.1:8081)投票时的term(3),如果,小于当前节点(127.0.0.1:8083),任期只能比当前节点大
			if (request.getTerm() < this.currTerm) {
				LOG.info("Node {} ignore PreVoteRequest from {}, term={}, currTerm={}.", getNodeId(),
					request.getServerId(), request.getTerm(), this.currTerm);
				// A follower replicator may not be started when this node become leader, so we must check it.
				checkReplicator(candidateId);
				break;
			}
			
			// 这一步骤先忽略不管,因为:ReplicatorGroup没有剖析.
			// A follower replicator may not be started when this node become leader, so we must check it.
			// check replicator state
			checkReplicator(candidateId);

			doUnlock = false;
			this.writeLock.unlock();

			// 加载本地的日志文件,获得最后一条log id.
			final LogId lastLogId = this.logManager.getLastLogId(true);

			doUnlock = true;
			this.writeLock.lock();
			// 解析远程节点传过来的lastLogIndex和lastLogTerm,并转换成:LogId
			final LogId requestLastLogId = new LogId(request.getLastLogIndex(), request.getLastLogTerm());
			
			
			// ********************************************************************************
			// 重点: 
			// 当远程节点(比如:127.0.0.1:8081)发起的投票请求参数(term/lastLogIndex) > 本节点(比如:127.0.0.1:8083),才允许授权.  
			// ********************************************************************************
			granted = requestLastLogId.compareTo(lastLogId) >= 0;

			LOG.info(
				"Node {} received PreVoteRequest from {}, term={}, currTerm={}, granted={}, requestLastLogId={}, lastLogId={}.",
				getNodeId(), request.getServerId(), request.getTerm(), this.currTerm, granted, requestLastLogId,
				lastLogId);
		} while (false);

		// 返回是否投票通过结果
		return RequestVoteResponse.newBuilder() //
			.setTerm(this.currTerm) //
			.setGranted(granted) //
			.build();
	} finally {
		if (doUnlock) {
			this.writeLock.unlock();
		}
	}
}
```
### (9). 总结
> 服务端处理预投票请求大体流程如下:  
> 1. 从远程节点(比如:127.0.0.1:8081)的请求参数里获得peerId(比如:127.0.0.1:8083).  
> 2. 根据peerId(比如:127.0.0.1:8083),从NodeManager单例对象里获得:Node对象实例,注意:Node是RaftServerService的实现类.  
> 3. 调用:RaftServerService.handlePreVoteRequest方法进行处理预投票.  
> 4. 投票通过的前提是: 远程节点(比如:127.0.0.1:8081)发起投票请求的投票参数(LastLogIndex/LastLogTerm)必须要大于本地节点(比如:127.0.0.1:8083)序列化到磁盘上的日志里的信息(LastLogIndex/LastLogTerm),投票才能算是通过.  