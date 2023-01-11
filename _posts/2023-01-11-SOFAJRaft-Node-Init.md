---
layout: post
title: 'SOFAJRaft源码之Node初始化(二二)' 
date: 2023-01-11
author: 李新
tags:  SOFAJRaft
---

### (1). 概述
一直以为Node的角色是网络通信的Server,但是,稍微的剖析了一下之后,发现好像,它在网络通信层属于Client端. 

### (2). Node UML图解
!["Node UML图解"](/assets/jraft/imgs/Node-ClassDiagram.jpg)

### (3). Scheduler初始化
```
// RaftTimerFactory(工厂模式)
this.timerManager = TIMER_FACTORY.getRaftScheduler(this.options.isSharedTimerPool(), this.options.getTimerPoolSize(), "JRaft-Node-ScheduleThreadPool");
```
### (4). 定时任务初始化
```
// Init timers
final String suffix = getNodeId().toString();
String name = "JRaft-VoteTimer-" + suffix;
this.voteTimer = new RepeatedTimer(name, this.options.getElectionTimeoutMs(), TIMER_FACTORY.getVoteTimer(
	this.options.isSharedVoteTimer(), name)) {

	@Override
	protected void onTrigger() {
		handleVoteTimeout();
	}

	@Override
	protected int adjustTimeout(final int timeoutMs) {
		return randomTimeout(timeoutMs);
	}
};

// 选举
name = "JRaft-ElectionTimer-" + suffix;
this.electionTimer = new RepeatedTimer(name, this.options.getElectionTimeoutMs(),
	TIMER_FACTORY.getElectionTimer(this.options.isSharedElectionTimer(), name)) {

	@Override
	protected void onTrigger() {
		handleElectionTimeout();
	}

	@Override
	protected int adjustTimeout(final int timeoutMs) {
		return randomTimeout(timeoutMs);
	}
};

// stop
name = "JRaft-StepDownTimer-" + suffix;
this.stepDownTimer = new RepeatedTimer(name, this.options.getElectionTimeoutMs() >> 1,
	TIMER_FACTORY.getStepDownTimer(this.options.isSharedStepDownTimer(), name)) {

	@Override
	protected void onTrigger() {
		handleStepDownTimeout();
	}
};

// Snapshot
name = "JRaft-SnapshotTimer-" + suffix;
this.snapshotTimer = new RepeatedTimer(name, this.options.getSnapshotIntervalSecs() * 1000,
	TIMER_FACTORY.getSnapshotTimer(this.options.isSharedSnapshotTimer(), name)) {

	private volatile boolean firstSchedule = true;

	@Override
	protected void onTrigger() {
		handleSnapshotTimeout();
	}

	@Override
	protected int adjustTimeout(final int timeoutMs) {
		if (!this.firstSchedule) {
			return timeoutMs;
		}

		// Randomize the first snapshot trigger timeout
		this.firstSchedule = false;
		if (timeoutMs > 0) {
			int half = timeoutMs / 2;
			return half + ThreadLocalRandom.current().nextInt(half);
		} else {
			return timeoutMs;
		}
	}
};
```
### (5). Disruptor初始化
```
// Disruptor
this.applyDisruptor = DisruptorBuilder.<LogEntryAndClosure> newInstance() //
	.setRingBufferSize(this.raftOptions.getDisruptorBufferSize()) //
	.setEventFactory(new LogEntryAndClosureFactory()) //
	.setThreadFactory(new NamedThreadFactory("JRaft-NodeImpl-Disruptor-", true)) //
	.setProducerType(ProducerType.MULTI) //
	.setWaitStrategy(new BlockingWaitStrategy()) //
	.build();
this.applyDisruptor.handleEventsWith(new LogEntryAndClosureHandler());
this.applyDisruptor.setDefaultExceptionHandler(new LogExceptionHandler<Object>(getClass().getSimpleName()));
this.applyQueue = this.applyDisruptor.start();
if (this.metrics.getMetricRegistry() != null) {
	this.metrics.getMetricRegistry().register("jraft-node-impl-disruptor",
		new DisruptorMetricSet(this.applyQueue));
}
```
### (6). FSMCaller初始化
```
// FSM有限状态机的构建
this.fsmCaller = new FSMCallerImpl();
```
### (7). LogManager初始化
```
private boolean initLogStorage() {
	Requires.requireNonNull(this.fsmCaller, "Null fsm caller");
	this.logStorage = this.serviceFactory.createLogStorage(this.options.getLogUri(), this.raftOptions);
	this.logManager = new LogManagerImpl();
	final LogManagerOptions opts = new LogManagerOptions();
	opts.setGroupId(this.groupId);
	opts.setLogEntryCodecFactory(this.serviceFactory.createLogEntryCodecFactory());
	opts.setLogStorage(this.logStorage);
	opts.setConfigurationManager(this.configManager);
	opts.setFsmCaller(this.fsmCaller);
	opts.setNodeMetrics(this.metrics);
	opts.setDisruptorBufferSize(this.raftOptions.getDisruptorBufferSize());
	opts.setRaftOptions(this.raftOptions);
	return this.logManager.init(opts);
}
```
### (8). RaftMetaStorage初始化
```
private boolean initMetaStorage() {
	this.metaStorage = this.serviceFactory.createRaftMetaStorage(this.options.getRaftMetaUri(), this.raftOptions);
	RaftMetaStorageOptions opts = new RaftMetaStorageOptions();
	opts.setNode(this);
	if (!this.metaStorage.init(opts)) {
		LOG.error("Node {} init meta storage failed, uri={}.", this.serverId, this.options.getRaftMetaUri());
		return false;
	}
	this.currTerm = this.metaStorage.getTerm();
	this.votedId = this.metaStorage.getVotedFor().copy();
	return true;
}
```
### (9). FSMCaller初始化
```
private boolean initFSMCaller(final LogId bootstrapId) {
	if (this.fsmCaller == null) {
		LOG.error("Fail to init fsm caller, null instance, bootstrapId={}.", bootstrapId);
		return false;
	}
	this.closureQueue = new ClosureQueueImpl(this.groupId);
	final FSMCallerOptions opts = new FSMCallerOptions();
	opts.setAfterShutdown(status -> afterShutdown());
	opts.setLogManager(this.logManager);
	opts.setFsm(this.options.getFsm());
	opts.setClosureQueue(this.closureQueue);
	opts.setNode(this);
	opts.setBootstrapId(bootstrapId);
	opts.setDisruptorBufferSize(this.raftOptions.getDisruptorBufferSize());
	return this.fsmCaller.init(opts);
}
```
### (10). BallotBox初始化
```
// 投票器构建
this.ballotBox = new BallotBox();
final BallotBoxOptions ballotBoxOpts = new BallotBoxOptions();
// 配置有限状态机
ballotBoxOpts.setWaiter(this.fsmCaller);
// 配置队列
ballotBoxOpts.setClosureQueue(this.closureQueue);
// 初始化日志投票器
if (!this.ballotBox.init(ballotBoxOpts)) {
	LOG.error("Node {} init ballotBox failed.", getNodeId());
	return false;
}
```
### (11). SnapshotExecutor初始化
```
private boolean initSnapshotStorage() {
	if (StringUtils.isEmpty(this.options.getSnapshotUri())) {
		LOG.warn("Do not set snapshot uri, ignore initSnapshotStorage.");
		return true;
	}
	this.snapshotExecutor = new SnapshotExecutorImpl();
	final SnapshotExecutorOptions opts = new SnapshotExecutorOptions();
	opts.setUri(this.options.getSnapshotUri());
	opts.setFsmCaller(this.fsmCaller);
	opts.setNode(this);
	opts.setLogManager(this.logManager);
	opts.setAddr(this.serverId != null ? this.serverId.getEndpoint() : null);
	opts.setInitTerm(this.currTerm);
	opts.setFilterBeforeCopyRemote(this.options.isFilterBeforeCopyRemote());
	// get snapshot throttle
	opts.setSnapshotThrottle(this.options.getSnapshotThrottle());
	return this.snapshotExecutor.init(opts);
}
```
### (12). ReplicatorGroup和RaftClientService初始化
```
this.replicatorGroup = new ReplicatorGroupImpl();
// ************************************************************************************
// 注意:这里初始化的实际是网络通端的Client来着的. 
// ************************************************************************************
this.rpcService = new DefaultRaftClientService(this.replicatorGroup, this.options.getAppendEntriesExecutors());
final ReplicatorGroupOptions rgOpts = new ReplicatorGroupOptions();
rgOpts.setHeartbeatTimeoutMs(heartbeatTimeout(this.options.getElectionTimeoutMs()));
rgOpts.setElectionTimeoutMs(this.options.getElectionTimeoutMs());
rgOpts.setLogManager(this.logManager);
rgOpts.setBallotBox(this.ballotBox);
rgOpts.setNode(this);
rgOpts.setRaftRpcClientService(this.rpcService);
rgOpts.setSnapshotStorage(this.snapshotExecutor != null ? this.snapshotExecutor.getSnapshotStorage() : null);
rgOpts.setRaftOptions(this.raftOptions);
rgOpts.setTimerManager(this.timerManager);

this.options.setMetricRegistry(this.metrics.getMetricRegistry());

if (!this.rpcService.init(this.options)) {
	LOG.error("Fail to init rpc service.");
	return false;
}
this.replicatorGroup.init(new NodeId(this.groupId, this.serverId), rgOpts);
```
### (13). ReadOnlyService初始化
```
this.readOnlyService = new ReadOnlyServiceImpl();
final ReadOnlyServiceOptions rosOpts = new ReadOnlyServiceOptions();
rosOpts.setFsmCaller(this.fsmCaller);
rosOpts.setNode(this);
rosOpts.setRaftOptions(this.raftOptions);

if (!this.readOnlyService.init(rosOpts)) {
	LOG.error("Fail to init readOnlyService.");
	return false;
}
```
### (14). 快照线程启动
```
if (this.snapshotExecutor != null && this.options.getSnapshotIntervalSecs() > 0) {
	LOG.debug("Node {} start snapshot timer, term={}.", getNodeId(), this.currTerm);
	this.snapshotTimer.start();
}
```
### (15). 总结 
Node的初始化还是比较复杂的,但是,Node所依赖的对象,我们全都源码剖析过了的,所以,应该不难理解. 