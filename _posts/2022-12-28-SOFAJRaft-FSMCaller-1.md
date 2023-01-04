---
layout: post
title: 'SOFAJRaft源码之FSMCaller上篇(十)' 
date: 2022-12-28
author: 李新
tags:  SOFAJRaft
---

### (1). 概述
本来这一节,要剖析SnapshotExecutor,结果,我发现:SnapshotExecutor底层却依赖:FSMCaller,所以,先剖析:FSMCaller
### (2). FSMCaller UML图

!["FSMCaller UML图"](/assets/jraft/imgs/FSMCaller-ClassDiagram.jpg)

### (3). FSMCaller接口签名
> 注意,注意,注意,细看FSMCaller的接口签名,你会发现:应用任务(apply)这个方法,在FSMCaller里是没有的.  

```
public interface FSMCaller extends Lifecycle<FSMCallerOptions>, Describer {

	interface LastAppliedLogIndexListener {
	        void onApplied(final long lastAppliedLogIndex);
	} // end LastAppliedLogIndexListener
	
	boolean isRunningOnFSMThread();
	
	
	void addLastAppliedLogIndexListener(final LastAppliedLogIndexListener listener);

	boolean onCommitted(final long committedIndex);
	
	boolean onSnapshotLoad(final LoadSnapshotClosure done);
	
	public void onSnapshotSaveSync(SaveSnapshotClosure done);
	
	boolean onSnapshotSave(final SaveSnapshotClosure done);
	
	boolean onLeaderStop(final Status status);

	boolean onLeaderStart(final long term);
	
	boolean onStartFollowing(final LeaderChangeContext ctx);

	boolean onStopFollowing(final LeaderChangeContext ctx);
	
	boolean onError(final RaftException error);
	
	long getLastAppliedIndex();
	
	long getLastCommittedIndex();
	
	void join() throws InterruptedException;
}
```
### (4). FSMCaller依赖关系
```
private volatile Thread                                         fsmThread;
// ************************************************************************************
// 日志管理
// ************************************************************************************
private LogManager                                              logManager;

// ************************************************************************************
// 状态机
// ************************************************************************************
private StateMachine                                            fsm;
private ClosureQueue                                            closureQueue;

// ************************************************************************************
// 最后提交给状态机的id
// ************************************************************************************
private final AtomicLong                                        lastAppliedIndex;

// ************************************************************************************
// 日志最后提交的id
// ************************************************************************************
private final AtomicLong                                        lastCommittedIndex;

// ************************************************************************************
// 最后提交给状态机的term
// ************************************************************************************
private long                                                    lastAppliedTerm;
private Closure                                                 afterShutdown;

// ************************************************************************************
// node里hold有网络通信
// ************************************************************************************
private NodeImpl                                                node;


private volatile TaskType                                       currTask;
private final AtomicLong                                        applyingIndex;
private volatile RaftException                                  error;

// ************************************************************************************
// 最后所有的业务操作,都转换成task,扔给了Disruptor
// ************************************************************************************
private Disruptor<ApplyTask>                                    disruptor;
private RingBuffer<ApplyTask>                                   taskQueue;
private volatile CountDownLatch                                 shutdownLatch;
private NodeMetrics                                             nodeMetrics;

// 扔给业务的一个应用监听器而已
private final CopyOnWriteArrayList<LastAppliedLogIndexListener> lastAppliedLogIndexListeners = new CopyOnWriteArrayList<>();
```
### (5). FSMCaller构造器
```
 public FSMCallerImpl() {
	super();
	// ************************************************************************************
	// 配置当前的任务为空闲
	// ************************************************************************************
	this.currTask = TaskType.IDLE;
	this.lastAppliedIndex = new AtomicLong(0);
	this.applyingIndex = new AtomicLong(0);
	this.lastCommittedIndex = new AtomicLong(0);
}
```
### (6). FSMCaller初始化
> FSMCaller初始化时,依赖如下对象:Node/LogManager/StateMachine 

```
public boolean init(final FSMCallerOptions opts) {
	this.logManager = opts.getLogManager();
	this.fsm = opts.getFsm();
	this.closureQueue = opts.getClosureQueue();
	this.afterShutdown = opts.getAfterShutdown();
	this.node = opts.getNode();
	this.nodeMetrics = this.node.getNodeMetrics();
	this.lastCommittedIndex.set(opts.getBootstrapId().getIndex());
	this.lastAppliedIndex.set(opts.getBootstrapId().getIndex());

	// 触发所有的:LastAppliedLogIndexListener事件
	notifyLastAppliedIndexUpdated(this.lastAppliedIndex.get());
	this.lastAppliedTerm = opts.getBootstrapId().getTerm();

	// *******************************************************************************************
	// 这部份的代码是:Disruptor的初始化(事件工厂/缓冲行大小/消费等待策略)
	// *******************************************************************************************
	this.disruptor = DisruptorBuilder.<ApplyTask> newInstance() //
		.setEventFactory(new ApplyTaskFactory()) // 配置事件工厂
		.setRingBufferSize(opts.getDisruptorBufferSize()) //
		.setThreadFactory(new NamedThreadFactory("JRaft-FSMCaller-Disruptor-", true)) //
		.setProducerType(ProducerType.MULTI) //
		.setWaitStrategy(new BlockingWaitStrategy()) // 配置消费等待策略为阻塞
		.build();

    // *******************************************************************************************
    // 配置Disruptor事件的消费类
	// *******************************************************************************************
	this.disruptor.handleEventsWith(new ApplyTaskHandler());
	// 配置Disruptor异常处理
	this.disruptor.setDefaultExceptionHandler(new LogExceptionHandler<Object>(getClass().getSimpleName()));
	// 启动Disruptor,并Hold住这个缓存队列.
	this.taskQueue = this.disruptor.start();

	// 监控数据
	if (this.nodeMetrics.getMetricRegistry() != null) {
		this.nodeMetrics.getMetricRegistry().register("jraft-fsm-caller-disruptor",
			new DisruptorMetricSet(this.taskQueue));
	}

	this.error = new RaftException(EnumOutter.ErrorType.ERROR_TYPE_NONE);
	LOG.info("Starts FSMCaller successfully.");
	return true;
}
```
### (7). TaskType
```
private enum TaskType {
	IDLE, //
	COMMITTED, //
	SNAPSHOT_SAVE, //
	SNAPSHOT_LOAD, //
	LEADER_STOP, //
	LEADER_START, //
	START_FOLLOWING, //
	STOP_FOLLOWING, //
	SHUTDOWN, //
	FLUSH, //
	ERROR;
}		
```
### (8).  ApplyTask
```
// 业务模型定义
private static class ApplyTask {
	TaskType            type;
	// union fields
	long                committedIndex;
	long                term;
	Status              status;
	LeaderChangeContext leaderChangeCtx;
	Closure             done;
	CountDownLatch      shutdownLatch;

	public void reset() {
		this.type = null;
		this.committedIndex = 0;
		this.term = 0;
		this.status = null;
		this.leaderChangeCtx = null;
		this.done = null;
		this.shutdownLatch = null;
	}
}
```
### (9). ApplyTaskFactory
```
// ********************************************************************************************
// 要求实现com.lmax.disruptor.EventFactory
// 固名思义:就是产生事件的工厂,这里的事件就是我们的业务模型
// ********************************************************************************************
private static class ApplyTaskFactory implements com.lmax.disruptor.EventFactory<ApplyTask> {

	@Override
	public ApplyTask newInstance() {
		return new ApplyTask();
	}
}
```
### (10). ApplyTaskHandler
```
// ********************************************************************************************
// 消费者消费时的回调函数
// 要求实现:com.lmax.disruptor.EventHandler
// ********************************************************************************************
private class ApplyTaskHandler implements com.lmax.disruptor.EventHandler<ApplyTask> {
	boolean      firstRun          = true;
	// max committed index in current batch, reset to -1 every batch
	private long maxCommittedIndex = -1;

	// ********************************************************************************************
	// Disruptor回调函数
	// ********************************************************************************************
	@Override
	public void onEvent(final ApplyTask event, final long sequence, final boolean endOfBatch) throws Exception {
		setFsmThread();
		// ********************************************************************************************
		// 委派给了:runApplyTask方法
		// 这里忽略,留到下一篇进行分析
		// ********************************************************************************************
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
### (11). FSMCallerImpl.enqueueTask
```
private boolean enqueueTask(final EventTranslator<ApplyTask> tpl) {
	// ... ...
	// *****************************************************************************
	// 向Disruptor发布事件
	// *****************************************************************************
	this.taskQueue.publishEvent(tpl);
	return true;
}
```
### (12). FSMCallerImpl常用方法分析
```
public boolean onSnapshotLoad(final LoadSnapshotClosure done) {
	// ****************************************************************************
	// 对于Lambda表达式,你可以理解成是对com.lmax.disruptor.EventTranslator的实现.如果难理解,你就理解成:
	// new EventTranslator<ApplyTask>(){  
	//    void translateTo(ApplyTask task, long sequence) { 
	//	       task.type = TaskType.SNAPSHOT_LOAD;
	//         task.done = done;
	//    }	
	// }
	// 到最后,始终是委派给上了上面分析的:enqueueTask方法,而enqueueTask方法是往Disruptor里发布事件而已
	// ****************************************************************************
	return enqueueTask((task, sequence) -> {
		task.type = TaskType.SNAPSHOT_LOAD;
		task.done = done;
	});
} // end 

public boolean onSnapshotSave(final SaveSnapshotClosure done) {
	// ****************************************************************************
	// 对于Lambda表达式,你可以理解成是对com.lmax.disruptor.EventTranslator的实现.如果难理解,你就理解成:
	// new EventTranslator<ApplyTask>(){  
	//    void translateTo(ApplyTask task, long sequence) { 
	//	       task.type = TaskType.SNAPSHOT_SAVE;
	//         task.done = done;
	//    }	
	// }
	// 到最后,始终是委派给上了上面分析的:enqueueTask方法,而enqueueTask方法是往Disruptor里发布事件而已
	// ****************************************************************************
	return enqueueTask((task, sequence) -> {
		task.type = TaskType.SNAPSHOT_SAVE;
		task.done = done;
	});
} // end 

public boolean onLeaderStop(final Status status) {
	// ****************************************************************************
	// 对于Lambda表达式,你可以理解成是对com.lmax.disruptor.EventTranslator的实现.如果难理解,你就理解成:
	// new EventTranslator<ApplyTask>(){  
	//    void translateTo(ApplyTask task, long sequence) { 
	//	       task.type = TaskType.LEADER_STOP;
	//         task.status = new Status(status);
	//    }	
	// }
	// 到最后,始终是委派给上了上面分析的:enqueueTask方法,而enqueueTask方法是往Disruptor里发布事件而已
	// ****************************************************************************
	return enqueueTask((task, sequence) -> {
		task.type = TaskType.LEADER_STOP;
		task.status = new Status(status);
	});
} // end

public boolean onLeaderStart(final long term) {
	// ****************************************************************************
	// 对于Lambda表达式,你可以理解成是对com.lmax.disruptor.EventTranslator的实现.如果难理解,你就理解成:
	// new EventTranslator<ApplyTask>(){  
	//    void translateTo(ApplyTask task, long sequence) { 
	//	       task.type = TaskType.LEADER_START;
	//         task.term = term;
	//    }	
	// }
	// 到最后,始终是委派给上了上面分析的:enqueueTask方法,而enqueueTask方法是往Disruptor里发布事件而已
	// ****************************************************************************
	return enqueueTask((task, sequence) -> {
		task.type = TaskType.LEADER_START;
		task.term = term;
	});
} // end 

public boolean onStartFollowing(final LeaderChangeContext ctx) {
	// ****************************************************************************
	// 对于Lambda表达式,你可以理解成是对com.lmax.disruptor.EventTranslator的实现.如果难理解,你就理解成:
	// new EventTranslator<ApplyTask>(){  
	//    void translateTo(ApplyTask task, long sequence) { 
	//	       task.type = TaskType.START_FOLLOWING;
	//         task.leaderChangeCtx = new LeaderChangeContext(ctx.getLeaderId(), ctx.getTerm(), ctx.getStatus());
	//    }	
	// }
	// 到最后,始终是委派给上了上面分析的:enqueueTask方法,而enqueueTask方法是往Disruptor里发布事件而已
	// ****************************************************************************
	return enqueueTask((task, sequence) -> {
		task.type = TaskType.START_FOLLOWING;
		task.leaderChangeCtx = new LeaderChangeContext(ctx.getLeaderId(), ctx.getTerm(), ctx.getStatus());
	});
} // end

public boolean onStopFollowing(final LeaderChangeContext ctx) {
	// ****************************************************************************
	// 对于Lambda表达式,你可以理解成是对com.lmax.disruptor.EventTranslator的实现.如果难理解,你就理解成:
	// new EventTranslator<ApplyTask>(){  
	//    void translateTo(ApplyTask task, long sequence) { 
	//	       task.type = TaskType.STOP_FOLLOWING;
	//         task.leaderChangeCtx = new LeaderChangeContext(ctx.getLeaderId(), ctx.getTerm(), ctx.getStatus());
	//    }	
	// }
	// 到最后,始终是委派给上了上面分析的:enqueueTask方法,而enqueueTask方法是往Disruptor里发布事件而已
	// ****************************************************************************
	return enqueueTask((task, sequence) -> {
		task.type = TaskType.STOP_FOLLOWING;
		task.leaderChangeCtx = new LeaderChangeContext(ctx.getLeaderId(), ctx.getTerm(), ctx.getStatus());
	});
} // end 
```
### (13). 总结
针对FSMCaller的大量API,实际底层最后是转换成:ApplyTask,并提交给:Disruptor异步处理. 