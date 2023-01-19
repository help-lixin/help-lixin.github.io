---
layout: post
title: 'SOFAJRaft源码之Node发起预投票请求(二十四)' 
date: 2023-01-11
author: 李新
tags:  SOFAJRaft
---

### (1). 概述
前面剖析了Node的启动方法,会判断是否有其它节点,如果有其它节点的情况下,会判断是否要进行选举,这一小篇,剖析下定时任务中的选举处理. 

### (2). Node预投票注意属性

```
// 预投票处理
private final Ballot                                                   prevVoteCtx              = new Ballot();

// Client
private RaftClientService                                              rpcService;
```
### (3). Node.init
```
// ... ...
// 定义选举定时任务
name = "JRaft-ElectionTimer-" + suffix;
this.electionTimer = new RepeatedTimer( 
							   name, 
							   this.options.getElectionTimeoutMs(), 
							   TIMER_FACTORY.getElectionTimer(this.options.isSharedElectionTimer(), name)
							   ) {
		@Override
		protected void onTrigger() {
			// ***********************************************************************
			// 处理投票超时
			// ***********************************************************************
			handleElectionTimeout();
		}

		@Override
		protected int adjustTimeout(final int timeoutMs) {
			return randomTimeout(timeoutMs);
		}
};


// ... ...

// 在Node初始化时,就把状态设置成:STATE_FOLLOWER.
// set state to follower
this.state = State.STATE_FOLLOWER;
```

### (4). Node.handleElectionTimeout
```
private void handleElectionTimeout() {
	// ... ...
	// RAFT要求,节点启动时是FOLLOWER
	if (this.state != State.STATE_FOLLOWER) {
		return;
	} // end if
	
	// ... ...
	// *************************************************************************
	// 准备投票
	// *************************************************************************
	preVote();
	// ... ...
}	
```
### (5).  Node.preVote
```
private void preVote() {
	// ... ... 
	
	// *******************************************************************************
	// 前面剖析过:Ballot会根据节点的数量,计算出法定票数.
	// *******************************************************************************
	// Ballot  prevVoteCtx = new Ballot();
	// this.conf.getConf() = 127.0.0.1:8081,127.0.0.1:8082,127.0.0.1:8083
	this.prevVoteCtx.init(this.conf.getConf(), this.conf.isStable() ? null : this.conf.getOldConf());
	
	// ************************************************************************************
	// 一个节点一个节点的连接,发起预投票请求
	// ************************************************************************************
	for (final PeerId peer : this.conf.listPeers()) {
		if (peer.equals(this.serverId)) {
			continue;
		}
		
		// ************************************************************************************
		// PingRequest(在这里,我忽略掉,不做分析了)
		// ************************************************************************************
		if (!this.rpcService.connect(peer.getEndpoint())) {
			LOG.warn("Node {} channel init failed, address={}.", getNodeId(), peer.getEndpoint());
			continue;
		}
		
		// 创建回调处理,
		final OnPreVoteRpcDone done = new OnPreVoteRpcDone(peer, this.currTerm);
		done.request = RequestVoteRequest.newBuilder() //
			.setPreVote(true) // it's a pre-vote request.
			.setGroupId(this.groupId) //
			.setServerId(this.serverId.toString()) //
			.setPeerId(peer.toString()) //
			.setTerm(this.currTerm + 1) // next term
			.setLastLogIndex(lastLogId.getIndex()) //
			.setLastLogTerm(lastLogId.getTerm()) //
			.build();
			
		// ************************************************************************************
		// 向服务器发起预投票请求
		// ************************************************************************************
		this.rpcService.preVote(peer.getEndpoint(), done.request, done);
	} // end for
	
	this.prevVoteCtx.grant(this.serverId);
	if (this.prevVoteCtx.isGranted()) { // 过半的节点预投票成功的情况下,进入投票阶段
		// ... ...
	} // end if
	
	// ... ... 
}	
```
### (6). Node.handlePreVoteResponse
```
public void handlePreVoteResponse(final PeerId peerId, final long term, final RequestVoteResponse response) {
	boolean doUnlock = true;
	this.writeLock.lock();
	try {
		if (this.state != State.STATE_FOLLOWER) { // 先验证,在投票之前,本节点的状态是否为:FOLLOWER
			LOG.warn("Node {} received invalid PreVoteResponse from {}, state not in STATE_FOLLOWER but {}.",
				getNodeId(), peerId, this.state);
			return;
		}

		if (term != this.currTerm) { // 验证任期
			LOG.warn("Node {} received invalid PreVoteResponse from {}, term={}, currTerm={}.", getNodeId(),
				peerId, term, this.currTerm);
			return;
		}

		// 如果其它节点(参与投票的节点)的任期,比当前节点的任期还要大
		if (response.getTerm() > this.currTerm) {
			LOG.warn("Node {} received invalid PreVoteResponse from {}, term {}, expect={}.", getNodeId(), peerId,
				response.getTerm(), this.currTerm);
			stepDown(response.getTerm(), false, new Status(RaftError.EHIGHERTERMRESPONSE,
				"Raft node receives higher term pre_vote_response."));
			return;
		}

		LOG.info("Node {} received PreVoteResponse from {}, term={}, granted={}.", getNodeId(), peerId, response.getTerm(), response.getGranted());
		// check granted quorum?
		if (response.getGranted()) { // 参与投票的节点,投了一票给本节点
			// *******************************************************************************
			// 接受,其余节点(Peer)处理预投票请求后,返回的信息后,进行预投票的判断,判断是否达到法定票数. 
			// *******************************************************************************
			this.prevVoteCtx.grant(peerId);
			if (this.prevVoteCtx.isGranted()) { // 只有达到了法定票数,才进入真正的选举.
				doUnlock = false;
				// *****************************************************************************
				// 开始选举,这里的内容,放到后面剖析.
				// *****************************************************************************
				electSelf();
			}
		}
	} finally {
		if (doUnlock) {
			this.writeLock.unlock();
		}
	}
}
```
### (7). 总结
> Node节点在初始化时会做以下几件事情:  
> 1. 本节点(Peer)为: 127.0.0.1:8081   
> 2. 其余节点(Peer)为: 127.0.0.1:8082,127.0.0.1:8083    
> 3. 从参数中,拿出所有的Peer(127.0.0.1:8081,127.0.0.1:8082,127.0.0.1:8083)   
> 4. 创建Ballot类,传入所有的Peer后,能自动计算出法定票数(quorum),注意:Ballot是Node的实例属性  
> 5. 遍历所有的Peer节点,发起心跳请求建立起连接   
> 6. 遍历所有的Peer节点,发起预投票请求(preVote)     
> 7. 其余节点(比如:127.0.0.1:8082)处理预投票请求(这个下一篇分析)   
> 8. 本节点处理投票后的结果(OnPreVoteRpcDone),实际就是调用:Ballot.grant方法,对法定票数(quorum)进行自减,当:法定票数为零时,代表多数节点已经确认通过.  