---
layout: post
title: 'SOFAJRaft源码之Node发起投票请求(二十六)' 
date: 2023-01-11
author: 李新
tags:  SOFAJRaft
---

### (1). 概述
前面,剖析了,预投票过程,预投票的过程是为了防止无效投票的,只有预投票之后通过之后,才能进入投票阶段,所以,这一篇主要剖析,发起投票请求的剖析. 

### (2). 入口在哪(NodeImpl.handlePreVoteResponse)
```
if (response.getGranted()) { // 参与投票的节点,投了一票给本节点
	this.prevVoteCtx.grant(peerId);
	if (this.prevVoteCtx.isGranted()) {
		doUnlock = false;
		// *******************************************************************************
		// 进行投票,判断投票是否通过.
		// *******************************************************************************
		electSelf();
	}
}
```
### (3). NodeImpl.electSelf
```
private void electSelf() {
	long oldTerm;
	try {
		LOG.info("Node {} start vote and grant vote self, term={}.", getNodeId(), this.currTerm);
		if (!this.conf.contains(this.serverId)) { // 验证节点是否在配置文件中存在.
			LOG.warn("Node {} can't do electSelf as it is not in {}.", getNodeId(), this.conf);
			return;
		}

		if (this.state == State.STATE_FOLLOWER) {
			LOG.debug("Node {} stop election timer, term={}.", getNodeId(), this.currTerm);
			// 在进入选举时,如果状态是:FOLLOWER状态,则先停止:预选举. 
			this.electionTimer.stop();
		}

		// 先忽略这部份内容
		// 重置Leader节点
		resetLeaderId(PeerId.emptyPeer(), new Status(RaftError.ERAFTTIMEDOUT, "A follower's leader_id is reset to NULL as it begins to request_vote."));
		
		// 在投票时,先让节点状态成为:CANDIDATE
		this.state = State.STATE_CANDIDATE;
		this.currTerm++;
		this.votedId = this.serverId.copy();
		LOG.debug("Node {} start vote timer, term={} .", getNodeId(), this.currTerm);

		// ********************************************************************
		// 启动投票线程
		// ********************************************************************
		this.voteTimer.start();
		// 初始化投票类和法定票数
		this.voteCtx.init(this.conf.getConf(), this.conf.isStable() ? null : this.conf.getOldConf());
		oldTerm = this.currTerm;
	} finally {
		this.writeLock.unlock();
	}

	final LogId lastLogId = this.logManager.getLastLogId(true);

	this.writeLock.lock();
	try {
		// vote need defense ABA after unlock&writeLock
		if (oldTerm != this.currTerm) {
			LOG.warn("Node {} raise term {} when getLastLogId.", getNodeId(), this.currTerm);
			return;
		}

		// 127.0.0.1:8081,127.0.0.1:8082,127.0.0.1:8083
		for (final PeerId peer : this.conf.listPeers()) {
			if (peer.equals(this.serverId)) { // 跳过本节点(127.0.0.1:8081)
				continue;
			}

			// 一个节点一个节点的遍历进行连接,连接不成功,直接跳过,进入下一次循环
			if (!this.rpcService.connect(peer.getEndpoint())) {
				LOG.warn("Node {} channel init failed, address={}.", getNodeId(), peer.getEndpoint());
				continue;
			}

			// 构建投票请求
			final OnRequestVoteRpcDone done = new OnRequestVoteRpcDone(peer, this.currTerm, this);
			done.request = RequestVoteRequest.newBuilder() //
				.setPreVote(false) // It's not a pre-vote request.
				.setGroupId(this.groupId) //
				.setServerId(this.serverId.toString()) //
				.setPeerId(peer.toString()) //
				.setTerm(this.currTerm) //
				.setLastLogIndex(lastLogId.getIndex()) //
				.setLastLogTerm(lastLogId.getTerm()) //
				.build();
			// *******************************************************************************
			// 向其余节点,发起投票请求
			// *******************************************************************************
			this.rpcService.requestVote(peer.getEndpoint(), done.request, done);
		}
		
		
		this.metaStorage.setTermAndVotedFor(this.currTerm, this.serverId);
		this.voteCtx.grant(this.serverId);
		
		if (this.voteCtx.isGranted()) {
			// ****************************************************************
			// 晋升为Leader
			// ****************************************************************
			becomeLeader();
		}
	} finally {
		this.writeLock.unlock();
	} // end finally
}
```
### (4). voteTimer初始化
```
// 这个是超时投票线程,会对状态进行降级处理.
this.voteTimer = new RepeatedTimer(name, this.options.getElectionTimeoutMs(), TIMER_FACTORY.getVoteTimer(this.options.isSharedVoteTimer(), name)) {

	@Override
	protected void onTrigger() {
		handleVoteTimeout();
	}

	@Override
	protected int adjustTimeout(final int timeoutMs) {
		return randomTimeout(timeoutMs);
	}
};
```
### (5). NodeImpl.handleVoteTimeout
```
private void handleVoteTimeout() {
	this.writeLock.lock();
	// 超时要求状态必须为:CANDIDATE
	if (this.state != State.STATE_CANDIDATE) {
		this.writeLock.unlock();
		return;
	}

	// 判断是否超时(估计在发心跳时,不停的更新这个状态)
	if (this.raftOptions.isStepDownWhenVoteTimedout()) {
		LOG.warn( "Candidate node {} term {} steps down when election reaching vote timeout: fail to get quorum vote-granted.",this.nodeId, this.currTerm);
		// **************************************************************************
		// 1. 设置节点关闭
		// **************************************************************************
		stepDown(this.currTerm, false, new Status(RaftError.ETIMEDOUT, "Vote timeout: fail to get quorum vote-granted."));
		// unlock in preVote
		
		// **************************************************************************
		// 2. 重新进入预投票
		// **************************************************************************
		preVote();
	} else {
		// **************************************************************************
		// 1. 安全选举
		// **************************************************************************
		LOG.debug("Node {} term {} retry to vote self.", getNodeId(), this.currTerm);
		// unlock in electSelf
		electSelf();
	}
}
```

### (6). 晋升为Leader(Node.becomeLeader)
```
private void becomeLeader() {
	Requires.requireTrue(this.state == State.STATE_CANDIDATE, "Illegal state: " + this.state);
	LOG.info("Node {} become leader of group, term={}, conf={}, oldConf={}.", getNodeId(), this.currTerm,
		this.conf.getConf(), this.conf.getOldConf());
	
	// 取消投票请求
	// cancel candidate vote timer
	stopVoteTimer();
	
	// 当前状态为Leader
	this.state = State.STATE_LEADER;
	this.leaderId = this.serverId.copy();
	
	// ReplicatorGroup重置term
	this.replicatorGroup.resetTerm(this.currTerm);
	
	
	// Start follower's replicators
	for (final PeerId peer : this.conf.listPeers()) {
		if (peer.equals(this.serverId)) { // 排除本机节点
			continue;
		}
		
		// ReplicatorGroup中添加节点
		LOG.debug("Node {} add a replicator, term={}, peer={}.", getNodeId(), this.currTerm, peer);
		if (!this.replicatorGroup.addReplicator(peer)) {
			LOG.error("Fail to add a replicator, peer={}.", peer);
		}
	}

	// Start learner's replicators
	for (final PeerId peer : this.conf.listLearners()) {
		LOG.debug("Node {} add a learner replicator, term={}, peer={}.", getNodeId(), this.currTerm, peer);
		if (!this.replicatorGroup.addReplicator(peer, ReplicatorType.Learner)) {
			LOG.error("Fail to add a learner replicator, peer={}.", peer);
		}
	}

	// init commit manager
	this.ballotBox.resetPendingIndex(this.logManager.getLastLogIndex() + 1);
	
	// Register _conf_ctx to reject configuration changing before the first log is committed.
	if (this.confCtx.isBusy()) {
		throw new IllegalStateException();
	}
	
	this.confCtx.flush(this.conf.getConf(), this.conf.getOldConf());
	this.stepDownTimer.start();
}
```
### (6). 总结
在正式发起投票前,要求有过半的节点预投票之后,才能进入投票过程,投票和预投票没有啥太大的区别,只有达到法定票数后,本节点才会晋升为Leader. 