---
layout: post
title: 'SOFAJRaft源码之Node投票请求处理(二十七)' 
date: 2023-01-11
author: 李新
tags:  SOFAJRaft
---

### (1). 概述
前面,剖析了,发起投票请求,这一篇主要剖析,接受投票请求.

### (2). NodeRequestProcessor.processRequest
```
public Message processRequest(final T request, final RpcRequestClosure done) {
	final PeerId peer = new PeerId();
	final String peerIdStr = getPeerId(request);
	if (peer.parse(peerIdStr)) {
		final String groupId = getGroupId(request);
		// ******************************************************************
		// 根据groupId和peer取出Node对象
		// ******************************************************************
		final Node node = NodeManager.getInstance().get(groupId, peer);
		if (node != null) {
			// ******************************************************************
			// 把node强制转换成:RaftServerService
			// ******************************************************************
			return processRequest0((RaftServerService) node, request, done);
		} else {
			// ... ...
		}
	} else {
		// ... ...
	}
}
```
### (3). RequestVoteRequestProcessor.processRequest0
```
public Message processRequest0(final RaftServerService service, final RequestVoteRequest request,
                                   final RpcRequestClosure done) {
	if (request.getPreVote()) { // 针对预投票处理
		return service.handlePreVoteRequest(request);
	} else {
		// *****************************************************************
		// 投票处理
		// *****************************************************************
		return service.handleRequestVoteRequest(request);
	} 
}
```
### (4). NodeImpl.handleRequestVoteRequest
```
public Message handleRequestVoteRequest(final RequestVoteRequest request) {
	boolean doUnlock = true;
	this.writeLock.lock();
	try {
		if (!this.state.isActive()) {
			LOG.warn("Node {} is not in active state, currTerm={}.", getNodeId(), this.currTerm);
			return RpcFactoryHelper //
				.responseFactory() //
				.newResponse(RequestVoteResponse.getDefaultInstance(), RaftError.EINVAL,
					"Node %s is not in active state, state %s.", getNodeId(), this.state.name());
		}
		
		final PeerId candidateId = new PeerId();
		if (!candidateId.parse(request.getServerId())) {
			LOG.warn("Node {} received RequestVoteRequest from {} serverId bad format.", getNodeId(), request.getServerId());
			return RpcFactoryHelper //
				.responseFactory() //
				.newResponse(RequestVoteResponse.getDefaultInstance(), RaftError.EINVAL,
					"Parse candidateId failed: %s.", request.getServerId());
		}

		// noinspection ConstantConditions
		do {
			// request = 127.0.0.1:8081
			// curr = 127.0.0.1:8082 
			// 检查,发起投票的节点(request)的任期 大于或者等于当前节点的任期
			// check term
			if (request.getTerm() >= this.currTerm) {
				LOG.info("Node {} received RequestVoteRequest from {}, term={}, currTerm={}.", getNodeId(), request.getServerId(), request.getTerm(), this.currTerm);
				// increase current term, change state to follower
				if (request.getTerm() > this.currTerm) {
					// 当节节点停止.
					stepDown(request.getTerm(), false, new Status(RaftError.EHIGHERTERMRESPONSE, "Raft node receives higher term RequestVoteRequest."));
				}
			} else {
				// ignore older term
				LOG.info("Node {} ignore RequestVoteRequest from {}, term={}, currTerm={}.", getNodeId(), request.getServerId(), request.getTerm(), this.currTerm);
				break;
			}
			
			doUnlock = false;
			this.writeLock.unlock();

			final LogId lastLogId = this.logManager.getLastLogId(true);

			doUnlock = true;
			this.writeLock.lock();
			
			// vote need ABA check after unlock&writeLock
			if (request.getTerm() != this.currTerm) {
				LOG.warn("Node {} raise term {} when get lastLogId.", getNodeId(), this.currTerm);
				break;
			}

			// 发起请求(request)的lastLogIndex和lastLogTerm > 当前节点的lastLogIndex和lastLogTerm
			final boolean logIsOk = new LogId(request.getLastLogIndex(), request.getLastLogTerm()).compareTo(lastLogId) >= 0;

			if (logIsOk && (this.votedId == null || this.votedId.isEmpty())) {
				stepDown(request.getTerm(), false, new Status(RaftError.EVOTEFORCANDIDATE, "Raft node votes for some candidate, step down to restart election_timer."));
				this.votedId = candidateId.copy();
				// 存储投票给哪个节点
				this.metaStorage.setVotedFor(candidateId);
			}
		} while (false);

		return RequestVoteResponse.newBuilder() //
			.setTerm(this.currTerm) //
			.setGranted(request.getTerm() == this.currTerm && candidateId.equals(this.votedId)) //
			.build();
	} finally {
		if (doUnlock) {
			this.writeLock.unlock();
		}
	}
}
```
### (5). 总结
服务端处理投票比较简单,只要发起请求的节点(比如:127.0.0.1:8081)的term和lastLogIndex比本地(比如:127.0.0.1:8082,127.0.0.1:8083)的大,即,授权通过. 