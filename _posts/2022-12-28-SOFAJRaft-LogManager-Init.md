---
layout: post
title: 'SOFAJRaft源码之LogManager初始化(四)' 
date: 2022-12-28
author: 李新
tags:  SOFAJRaft
---

### (1). 概述
还是按照官网的框架图,以局部作为点,进行剖析,这一小篇主要分析:LogManager,它是对LogStorage进行了包装,为什么要包装呢?因为,这个类它在LogStorage的基础上增加了:缓存和批量提交的功能,咱们一点点的剖析. 

### (2). LogManager UML图解
!["LogManager UML图"](/assets/jraft/imgs/LogManager-ClassDiagram.jpg)

### (3). LogManager初始化剖析(LogManager.init)
```
public boolean init(final LogManagerOptions opts) {
	this.writeLock.lock();
	try {
		if (opts.getLogStorage() == null) {
			LOG.error("Fail to init log manager, log storage is null");
			return false;
		}

		this.groupId = opts.getGroupId();
		this.raftOptions = opts.getRaftOptions();
		this.nodeMetrics = opts.getNodeMetrics();
		this.logStorage = opts.getLogStorage();
		this.configManager = opts.getConfigurationManager();

		LogStorageOptions lsOpts = new LogStorageOptions();
		lsOpts.setGroupId(opts.getGroupId());
		lsOpts.setConfigurationManager(this.configManager);
		lsOpts.setLogEntryCodecFactory(opts.getLogEntryCodecFactory());
		// **********************************************************************
		// LogStorage初始化
		// **********************************************************************
		if (!this.logStorage.init(lsOpts)) {
			LOG.error("Fail to init logStorage");
			return false;
		}

		// 1
		this.firstLogIndex = this.logStorage.getFirstLogIndex();
		// 0
		this.lastLogIndex = this.logStorage.getLastLogIndex();
		// hold住最后一个日志id
		this.diskId = new LogId(this.lastLogIndex, getTermFromLogStorage(this.lastLogIndex));
		this.fsmCaller = opts.getFsmCaller();

		// disuptor创建
		this.disruptor = DisruptorBuilder.<StableClosureEvent> newInstance() //
				// StableClosureEventFactory为Disruptor的工厂,要求实现:com.lmax.disruptor.EventFactory
				.setEventFactory(new StableClosureEventFactory()) //
				// 指定形型队列的大小(必须是2的指数)
				.setRingBufferSize(opts.getDisruptorBufferSize()) //
				// 自定义线程工厂
				.setThreadFactory(new NamedThreadFactory("JRaft-LogManager-Disruptor-", true)) //
				// 指定生产者是多线程模式还是单线程模式
				.setProducerType(ProducerType.MULTI) //
				// 配置消费者等待策略
				/*
				 *  Use timeout strategy in log manager. If timeout happens, it will called reportError to halt the node.
				 */
				.setWaitStrategy(new TimeoutBlockingWaitStrategy(
					this.raftOptions.getDisruptorPublishEventWaitTimeoutSecs(), TimeUnit.SECONDS)) //
				.build();
		// *******************************************************************
		// 配置Disruptor的消费者
		// *******************************************************************
		this.disruptor.handleEventsWith(new StableClosureEventHandler());

		// *******************************************************************
		// 配置Disruptor的异常处理
		// *******************************************************************
		this.disruptor.setDefaultExceptionHandler(new LogExceptionHandler<Object>(this.getClass().getSimpleName(),
				(event, ex) -> reportError(-1, "LogManager handle event error")));

		// 返回的实际是一个形式队列
		this.diskQueue = this.disruptor.start();

		// 监控处理
		if (this.nodeMetrics.getMetricRegistry() != null) {
			this.nodeMetrics.getMetricRegistry().register("jraft-log-manager-disruptor",
				new DisruptorMetricSet(this.diskQueue));
		}
	} finally {
		this.writeLock.unlock();
	}
	return true;
}
```
### (4). 总结
> LogManager初始化比较简单:   
> 1. 它(LogManager)需要LogStorage,所以,需要从外部传入.  
> 2. 内部使用了Disruptor来异步处理,由于需要传递一个:Closure,所以,异步之后,会通过Closure进行回调返回给调用方的. 