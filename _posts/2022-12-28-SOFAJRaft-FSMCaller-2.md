---
layout: post
title: 'SOFAJRaft源码之FSMCaller下篇(十一)' 
date: 2022-12-28
author: 李新
tags:  SOFAJRaft
---

### (1). 概述
在上一篇对FSMCaller的初始化和基本方法进行了剖析,发现,大量的API操作最终是往Disruptor里发布事件而已,所以,Disruptor的消费(ApplyTaskHandler)是这一篇要分析的. 

### (2). ApplyTaskHandler
```
private class ApplyTaskHandler implements EventHandler<ApplyTask> {
	boolean      firstRun          = true;
	// max committed index in current batch, reset to -1 every batch
	private long maxCommittedIndex = -1;

	@Override
	public void onEvent(final ApplyTask event, final long sequence, final boolean endOfBatch) throws Exception {
		setFsmThread();
		// ******************************************************************************************
		// 委派给了runApplyTask
		// ******************************************************************************************
		this.maxCommittedIndex = runApplyTask(event, this.maxCommittedIndex, endOfBatch);
	}

	private void setFsmThread() {
		if (firstRun) {
			fsmThread = Thread.currentThread();
			firstRun = false;
		}
	}
}
```
### (3). FSMCaller.runApplyTask
```
private long runApplyTask(final ApplyTask task, long maxCommittedIndex, final boolean endOfBatch) {
	CountDownLatch shutdown = null;
	if (task.type == TaskType.COMMITTED) {
		if (task.committedIndex > maxCommittedIndex) {
			maxCommittedIndex = task.committedIndex;
		}
		task.reset();
	} else {

		if (maxCommittedIndex >= 0) {
			this.currTask = TaskType.COMMITTED;
			doCommitted(maxCommittedIndex);
			maxCommittedIndex = -1L; // reset maxCommittedIndex
		}

		final long startMs = Utils.monotonicMs();
		try {
			switch (task.type) {
				case COMMITTED:
					Requires.requireTrue(false, "Impossible");
					break;
				case SNAPSHOT_SAVE:
				    //  ********************************************************************
					// 快照保存
					//  ********************************************************************
					onSnapshotSaveSync(task);
					break;
				case SNAPSHOT_LOAD:
					//  ********************************************************************
					// 快照加载
					//  ********************************************************************
					this.currTask = TaskType.SNAPSHOT_LOAD;
					if (passByStatus(task.done)) {
						doSnapshotLoad((LoadSnapshotClosure) task.done);
					}
					break;
				case LEADER_STOP:
					this.currTask = TaskType.LEADER_STOP;
					doLeaderStop(task.status);
					break;
				case LEADER_START:
					this.currTask = TaskType.LEADER_START;
					doLeaderStart(task.term);
					break;
				case START_FOLLOWING:
					this.currTask = TaskType.START_FOLLOWING;
					doStartFollowing(task.leaderChangeCtx);
					break;
				case STOP_FOLLOWING:
					this.currTask = TaskType.STOP_FOLLOWING;
					doStopFollowing(task.leaderChangeCtx);
					break;
				case ERROR:
					this.currTask = TaskType.ERROR;
					doOnError((OnErrorClosure) task.done);
					break;
				case IDLE:
					Requires.requireTrue(false, "Can't reach here");
					break;
				case SHUTDOWN:
					this.currTask = TaskType.SHUTDOWN;
					shutdown = task.shutdownLatch;
					break;
				case FLUSH:
					this.currTask = TaskType.FLUSH;
					shutdown = task.shutdownLatch;
					break;
			}
		} finally {
			this.nodeMetrics.recordLatency(task.type.metricName(), Utils.monotonicMs() - startMs);
			// 重置一个业务模型
			task.reset();
		}
	}


	try {
		if (endOfBatch && maxCommittedIndex >= 0) {
			this.currTask = TaskType.COMMITTED;
			doCommitted(maxCommittedIndex);
			maxCommittedIndex = -1L; // reset maxCommittedIndex
		}
		this.currTask = TaskType.IDLE;
		return maxCommittedIndex;
	} finally {
		if (shutdown != null) {
			shutdown.countDown();
		}
	}
}
```
### (4). FSMCaller.onSnapshotSaveSync
```
private void onSnapshotSaveSync(final ApplyTask task) {
	this.currTask = TaskType.SNAPSHOT_SAVE;
	if (passByStatus(task.done)) { // 验证下error是否有值,如果有内容的话就不能继续往下走了
		// *********************************************************************
		// FSMCaller.doSnapshotSave
		// *********************************************************************
		doSnapshotSave((SaveSnapshotClosure) task.done);
	}
}
```
### (5). FSMCaller.doSnapshotSave
```
private void doSnapshotSave(final SaveSnapshotClosure done) {
	Requires.requireNonNull(done, "SaveSnapshotClosure is null");
	final long lastAppliedIndex = this.lastAppliedIndex.get();
	
	// 创建快照存储元数据(SnapshotMeta)
	final RaftOutter.SnapshotMeta.Builder metaBuilder = RaftOutter.SnapshotMeta.newBuilder() //
		.setLastIncludedIndex(lastAppliedIndex) //
		.setLastIncludedTerm(this.lastAppliedTerm);
	final ConfigurationEntry confEntry = this.logManager.getConfiguration(lastAppliedIndex);
	if (confEntry == null || confEntry.isEmpty()) {
		LOG.error("Empty conf entry for lastAppliedIndex={}", lastAppliedIndex);
		ThreadPoolsFactory.runClosureInThread(getNode().getGroupId(), done, new Status(RaftError.EINVAL,
			"Empty conf entry for lastAppliedIndex=%s", lastAppliedIndex));
		return;
	}
	
	// 拷贝相关属性
	for (final PeerId peer : confEntry.getConf()) {
		metaBuilder.addPeers(peer.toString());
	}
	for (final PeerId peer : confEntry.getConf().getLearners()) {
		metaBuilder.addLearners(peer.toString());
	}
	if (confEntry.getOldConf() != null) {
		for (final PeerId peer : confEntry.getOldConf()) {
			metaBuilder.addOldPeers(peer.toString());
		}
		for (final PeerId peer : confEntry.getOldConf().getLearners()) {
			metaBuilder.addOldLearners(peer.toString());
		}
	}
	
	// ***********************************************************************
	// 从外部传入的:SaveSnapshotClosure里的start方法,获得:SnapshotWriter
	// ***********************************************************************
	final SnapshotWriter writer = done.start(metaBuilder.build());
	if (writer == null) {
		done.run(new Status(RaftError.EINVAL, "snapshot_storage create SnapshotWriter failed"));
		return;
	}
	
	// *********************************************************************
	// 委托给:StateMachine的onSnapshotSave方法
	// 状态机的方法,我留到下一篇去剖析.
	// *********************************************************************
	this.fsm.onSnapshotSave(writer, done);
}
```
### (6). FSMCaller.doSnapshotLoad
```
private void doSnapshotLoad(final LoadSnapshotClosure done) {
	Requires.requireNonNull(done, "LoadSnapshotClosure is null");
	
	// ******************************************************************************
	// 从外部传入的:LoadSnapshotClosure.start方法中,获得:SnapshotReader
	// ******************************************************************************
	final SnapshotReader reader = done.start();
	if (reader == null) { // 验证一把
		done.run(new Status(RaftError.EINVAL, "open SnapshotReader failed"));
		return;
	}

	// 通过:SnapshotReader读取元数据(SnapshotMeta)
	final RaftOutter.SnapshotMeta meta = reader.load();
	if (meta == null) { // 验证一把
		done.run(new Status(RaftError.EINVAL, "SnapshotReader load meta failed"));
		if (reader.getRaftError() == RaftError.EIO) {
			final RaftException err = new RaftException(EnumOutter.ErrorType.ERROR_TYPE_SNAPSHOT, RaftError.EIO,
				"Fail to load snapshot meta");
			setError(err);
		}
		return;
	}
    
	// 验证
	// 提交给状态机的日志不能比快照的日志信息还要新,如果是这样的话,代表快照内容是旧的,不应该加载
	final LogId lastAppliedId = new LogId(this.lastAppliedIndex.get(), this.lastAppliedTerm);
	final LogId snapshotId = new LogId(meta.getLastIncludedIndex(), meta.getLastIncludedTerm());
	if (lastAppliedId.compareTo(snapshotId) > 0) {
		done.run(new Status(
			RaftError.ESTALE,
			"Loading a stale snapshot last_applied_index=%d last_applied_term=%d snapshot_index=%d snapshot_term=%d",
			lastAppliedId.getIndex(), lastAppliedId.getTerm(), snapshotId.getIndex(), snapshotId.getTerm()));
		return;
	}

	// ******************************************************************************************
	// 委托给:StateMachine.onSnapshotLoad
	// StateMachine的内容,下一篇再分析.
	// ******************************************************************************************
	if (!this.fsm.onSnapshotLoad(reader)) {
		done.run(new Status(-1, "StateMachine onSnapshotLoad failed"));
		final RaftException e = new RaftException(EnumOutter.ErrorType.ERROR_TYPE_STATE_MACHINE,
			RaftError.ESTATEMACHINE, "StateMachine onSnapshotLoad failed");
		setError(e);
		return;
	}

	if (meta.getOldPeersCount() == 0) {
		// Joint stage is not supposed to be noticeable by end users.
		final Configuration conf = new Configuration();
		for (int i = 0, size = meta.getPeersCount(); i < size; i++) {
			final PeerId peer = new PeerId();
			Requires.requireTrue(peer.parse(meta.getPeers(i)), "Parse peer failed");
			conf.addPeer(peer);
		}
		this.fsm.onConfigurationCommitted(conf);
	}

	this.lastCommittedIndex.set(meta.getLastIncludedIndex());
	this.lastAppliedIndex.set(meta.getLastIncludedIndex());
	this.lastAppliedTerm = meta.getLastIncludedTerm();
	done.run(Status.OK());
}
```
### (7). 总结
在这一篇,对FSMCaller里的:onSnapshotSaveSync/onSnapshotLoad进行了剖析,内容比较简单,最终是委托给了:StateMachine进行操作,可以理解:FSMCaller是对StateMachine的包装,包括一些验证,以及记录一些信息(lastCommittedIndex/lastAppliedIndex/lastAppliedTerm). 