---
layout: post
title: 'SOFAJRaft源码之BallotBox之日志确认(十七)' 
date: 2022-12-28
author: 李新
tags:  SOFAJRaft
---

### (1). 概述
在前面对BallotBox追加任务进行了剖析,在这一部份,主要剖析日志确认这一部份,RAFT要求,过半加一的节点确认后,这条日志才能真正的成功,才能推动commitIndex和applyIndex. 

### (2).  BallotBoxTest
```
@Test
public void testCommitAt() {
	
	// *********************************************************	
	// 这一部份,在上一篇有剖析过了,就不粘源码了.
	// *********************************************************	
	// ... ...
	
	
	// ********************************************************************
	// 注意,注意,注意: 
	// 由于是三个节点,所以,需要2个节点确认,才会真正的确认来着的
	// ********************************************************************
	
	
	// ********************************************************************
	// 8081 节点确认日志复制完成
	// ********************************************************************
	assertTrue(this.box.commitAt(1, 1, new PeerId("localhost", 8081)));
	assertEquals(0, this.box.getLastCommittedIndex());
	assertEquals(1, this.box.getPendingIndex());
	
	// ********************************************************************
	// 8082 节点确认日志复制完成
	// ********************************************************************
	assertTrue(this.box.commitAt(1, 1, new PeerId("localhost", 8082)));
	assertEquals(1, this.box.getLastCommittedIndex());
	assertEquals(2, this.box.getPendingIndex());
}
```
### (3). BallotBox.commitAt
```
public boolean commitAt(final long firstLogIndex, final long lastLogIndex, final PeerId peer) {
	// TODO  use lock-free algorithm here?
	final long stamp = this.stampedLock.writeLock();
	// 定义临时:lastCommittedIndex
	long lastCommittedIndex = 0;
	try {
		// commitAt的目的是处理队列里的数据来着,pendingIndex=0,代表队列为空,所以,不需要往下走了. 
		if (this.pendingIndex == 0) {
			return false;
		}
		
		
		if (lastLogIndex < this.pendingIndex) {
			return true;
		}

		if (lastLogIndex >= this.pendingIndex + this.pendingMetaQueue.size()) {
			throw new ArrayIndexOutOfBoundsException();
		}

		final long startAt = Math.max(this.pendingIndex, firstLogIndex);
		Ballot.PosHint hint = new Ballot.PosHint();
		for (long logIndex = startAt; logIndex <= lastLogIndex; logIndex++) {
			// *********************************************************************
			// 这是个重点:Ballot对象是多线程共享的,多个节点反馈信息后,回调commitAt时,弹出的都是同一实例对象.
			// *********************************************************************
			final Ballot bl = this.pendingMetaQueue.get((int) (logIndex - this.pendingIndex));
			
			// *********************************************************************
			// 进行投票
			// *********************************************************************
			hint = bl.grant(peer, hint);
			
			// *********************************************************************
			// 验证投票(this.quorum <= 0 && this.oldQuorum <= 0;)
			// 要求投票后,quorum和oldQuorum结果必须是零,才对临时lastCommittedIndex进行赋值.
			// *********************************************************************
			if (bl.isGranted()) {
				// 这时候的:临时lastCommittedIndex切换成了:logIndex
				lastCommittedIndex = logIndex;
			}
		}
		
		// 当投票后,quorum不为零的情况下,临时lastCommittedIndex是没有变来着的,所以,这时候,不继续往下走了.
		if (lastCommittedIndex == 0) {
			return true;
		}
		
		// *******************************************************************************
		// 过半以上的节点确认之后(临时lastCommittedIndex>0),从队列里拿出稳除数据
		// 代表这条数据,已经多数节点复制成功了的
		// *******************************************************************************
		this.pendingMetaQueue.removeFromFirst((int) (lastCommittedIndex - this.pendingIndex) + 1);
		LOG.debug("Committed log fromIndex={}, toIndex={}.", this.pendingIndex, lastCommittedIndex);
		
		// *************************************************************
		// pendingIndex往前推进
		// *************************************************************
		this.pendingIndex = lastCommittedIndex + 1;
		this.lastCommittedIndex = lastCommittedIndex;
	} finally {
		this.stampedLock.unlockWrite(stamp);
	}
	
	// *************************************************************
	// 这部份,我们前面剖析过了:
	// 委派给FSMCaller进行提交,最终会调用到:StateMachine.onApply
	// *************************************************************
	this.waiter.onCommitted(lastCommittedIndex);
	return true;
}
```
### (4). Ballot.PosHint.grant
```
// Ballot 摘抄属性
private final List<UnfoundPeerId> peers    = new ArrayList<>();
private int                       quorum;

private final List<UnfoundPeerId> oldPeers = new ArrayList<>();
private int                       oldQuorum;


public PosHint grant(final PeerId peerId, final PosHint hint) {
	// ***************************************************************
	// peers处理
	// 遍历peers里所有的节点
	// ***************************************************************
	UnfoundPeerId peer = findPeer(peerId, this.peers, hint.pos0);
	if (peer != null) {
		if (!peer.found) {
			// 为节点的:found赋值
			peer.found = true;
			// ********************************************************************
			// 重点: 对投票数进行递减
			// ********************************************************************
			this.quorum--;
		}
		hint.pos0 = peer.index;
	} else {
		hint.pos0 = -1;
	}

	// ***************************************************************
	// oldPerrs处理
	// 遍历oldPerrs里所有的节点
	// ***************************************************************
	if (this.oldPeers.isEmpty()) {
		hint.pos1 = -1;
		return hint;
	}
	peer = findPeer(peerId, this.oldPeers, hint.pos1);
	if (peer != null) {
		if (!peer.found) {
			peer.found = true;
			// ********************************************************************
			// 重点: 对投票数进行递减
			// ********************************************************************
			this.oldQuorum--;
		}
		hint.pos1 = peer.index;
	} else {
		hint.pos1 = -1;
	}
	return hint;
}
```
### (5). 总结
我们可以理解成:BallotBox主要用于缓存日志请求,然后,等待过半加一的节点确认日志后,从缓存中剔除日志.  