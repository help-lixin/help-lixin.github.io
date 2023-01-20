---
layout: post
title: 'SOFAJRaft源码之Node提交任务(二十八)' 
date: 2023-01-11
author: 李新
tags:  SOFAJRaft
---

### (1). 概述
在这一篇,继续剖析完一个比较重要的环节,那就是应用任务,这是JRaft提供给业务开发人员的一个重要接口. 

### (2). NodeImpl.apply
> apply方法最终仅仅是把task组装成:LogEntry,并,提交给:disruptor,反而是要看disruptor是如何实现的了. 


```
public void apply(final Task task) {
	// 验证下是否关闭了
	if (this.shutdownLatch != null) {
		ThreadPoolsFactory.runClosureInThread(this.groupId, task.getDone(), new Status(RaftError.ENODESHUTDOWN, "Node is shutting down."));
		throw new IllegalStateException("Node is shutting down");
	}
	// 要求传入的task不能为空
	Requires.requireNonNull(task, "Null task");

	// **************************************************************************
	// 把从task里拿出data,传入到:LogEntry
	// **************************************************************************
	final LogEntry entry = new LogEntry();
	entry.setData(task.getData());

	// 创建:EventTranslator的实现,实际就是给:LogEntryAndClosure赋值.
	final EventTranslator<LogEntryAndClosure> translator = (event, sequence) -> {
	  event.reset();
	  event.done = task.getDone();
	  event.entry = entry;
	  event.expectedTerm = task.getExpectedTerm();
	};

	// ***************************************************************************
	// 不论任务模式是阻塞或非阻塞,都是往:disruptor里传数据.
	// ***************************************************************************
	switch(this.options.getApplyTaskMode()) {
	  case Blocking:
	     // ****************************************************************************
		 // 提交任务给:disruptor
		 // ****************************************************************************
		this.applyQueue.publishEvent(translator);
		break;
	  case NonBlocking:
	  default:
		  // ****************************************************************************
		  // 提交任务给:disruptor
		  // ****************************************************************************
		if (!this.applyQueue.tryPublishEvent(translator)) {
		  String errorMsg = "Node is busy, has too many tasks, queue is full and bufferSize="+ this.applyQueue.getBufferSize();
		  ThreadPoolsFactory.runClosureInThread(this.groupId, task.getDone(), new Status(RaftError.EBUSY, errorMsg));
		  LOG.warn("Node {} applyQueue is overload.", getNodeId());
		  
		  this.metrics.recordTimes("apply-task-overload-times", 1);
		  if(task.getDone() == null) {
			throw new OverloadException(errorMsg);
		  } // end if
		} // end if
		break;
	} // end switch
} // end apply
```
### (3). NodeImpl.init
> 在disruptor初始化时,会配置事件的处理,所以,我们主要剖析:LogEntryAndClosureHandler即可.

```
this.applyDisruptor.handleEventsWith(new LogEntryAndClosureHandler());
```
### (4). LogEntryAndClosureHandler.onEvent
```
@Override
public void onEvent(final LogEntryAndClosure event, final long sequence, final boolean endOfBatch)
																								  throws Exception {
	// ************************************************************************************
	// 如果,event里有配置shutdownLatch,代表任务是要立即执行的,否则,代表任务是要批量执行的.
	// ************************************************************************************
	if (event.shutdownLatch != null) { // 如果,event里的shutdownLatch不为空的情况下
		// 如果,缓冲列表不为空
		if (!this.tasks.isEmpty()) {
			// 判断应用任务
			executeApplyingTasks(this.tasks);

			// 由于event是交还给disruptor,所以,要把event全部属性给清空,还要把缓存列表(tasks)给清空.
			reset();
		}
		// 计数器进行计数
		final int num = GLOBAL_NUM_NODES.decrementAndGet();
		LOG.info("The number of active nodes decrement to {}.", num);
		event.shutdownLatch.countDown();
		return;
	}

	// 添加任务到任务列表
	this.tasks.add(event);
	// 如果任务达到了批(或者disruptor里列表结尾了),则批量应用任务
	if (this.tasks.size() >= NodeImpl.this.raftOptions.getApplyBatch() || endOfBatch) {
		executeApplyingTasks(this.tasks);
		reset();
	}
}
```
### (5). NodeImpl.executeApplyingTasks
```
private void executeApplyingTasks(final List<LogEntryAndClosure> tasks) {
	this.writeLock.lock();
	try {
		final int size = tasks.size();
		// ******************************************************************************
		// 针对Leader的处理.
		// ******************************************************************************
		if (this.state != State.STATE_LEADER) {
			final Status st = new Status();
			if (this.state != State.STATE_TRANSFERRING) {
				st.setError(RaftError.EPERM, "Is not leader.");
			} else {
				st.setError(RaftError.EBUSY, "Is transferring leadership.");
			}
			LOG.debug("Node {} can't apply, status={}.", getNodeId(), st);
			final List<Closure> dones = tasks.stream().map(ele -> ele.done)
					.filter(Objects::nonNull).collect(Collectors.toList());
			ThreadPoolsFactory.runInThread(this.groupId, () -> {
				for (final Closure done : dones) {
					done.run(st);
				}
			});
			return;
		}

		final List<LogEntry> entries = new ArrayList<>(size);
		for (int i = 0; i < size; i++) {
			final LogEntryAndClosure task = tasks.get(i);

			// 再次验证下:term是否相同.
			if (task.expectedTerm != -1 && task.expectedTerm != this.currTerm) {
				LOG.debug("Node {} can't apply task whose expectedTerm={} doesn't match currTerm={}.", getNodeId(),
					task.expectedTerm, this.currTerm);
				if (task.done != null) {
					final Status st = new Status(RaftError.EPERM, "expected_term=%d doesn't match current_term=%d",
						task.expectedTerm, this.currTerm);
					ThreadPoolsFactory.runClosureInThread(this.groupId, task.done, st);
					task.reset();
				}
				continue;
			}

			if (!this.ballotBox.appendPendingTask(this.conf.getConf(), this.conf.isStable() ? null : this.conf.getOldConf(), task.done)) {
				ThreadPoolsFactory.runClosureInThread(this.groupId, task.done, new Status(RaftError.EINTERNAL, "Fail to append task."));
				task.reset();
				continue;
			}

			// ***************************************************************************
			// 为任务配置:term/type
			// ***************************************************************************
			// set task entry info before adding to list.
			task.entry.getId().setTerm(this.currTerm);
			task.entry.setType(EnumOutter.EntryType.ENTRY_TYPE_DATA);

			// ***************************************************************************
			// 添加任务到集合里
			// ***************************************************************************
			entries.add(task.entry);
			task.reset();
		}

        // *****************************************************************************
		// 交给LogManager追加日志,追加日志成功后,
		// *****************************************************************************
		this.logManager.appendEntries(entries, new LeaderStableClosure(entries));

		// update conf.first
		checkAndSetConfiguration(true);
	} finally {
		this.writeLock.unlock();
	}
}
```
### (6). LeaderStableClosure
```
class LeaderStableClosure extends LogManager.StableClosure {

	public LeaderStableClosure(final List<LogEntry> entries) {
		super(entries);
	}

	@Override
	public void run(final Status status) {
		if (status.isOk()) {
			// ********************************************************************************
			// 提交日志,前面分析过了,就不提了.
			// ********************************************************************************
			NodeImpl.this.ballotBox.commitAt(this.firstLogIndex, this.firstLogIndex + this.nEntries - 1, NodeImpl.this.serverId);
		} else {
			LOG.error("Node {} append [{}, {}] failed, status={}.", getNodeId(), this.firstLogIndex, this.firstLogIndex + this.nEntries - 1, status);
		}
	} // end run
}
```
### (7). 总结
> apply方法的作用如下:  
> 1. 把task通过LogEntry进行包裹.   
> 2. 通过Disruptor提交任务.  
> 3. LogEntryAndClosureHandler接受处理任务.  
> 4. 在提交任务之前,验证是否为Leader/Term.  
> 5. 如果任务是需要立即给出反馈的(shutdownLatch),则,立即执行.  
> 6. 如果任务不是需要立即给出反馈的,则,会添加到集合中,进行批量提交.   
> 7. 最终任务会交给:LogManager进行任务的追加.   
> 8. 日志追加完毕后,会调用:LeaderStableClosure,而,LeaderStableClosure会委托给:BallotBox.  
> 9. BallotBox会要求"法定票数"的机器都成功之后,才能算成功.  
> 10. BallotBox会调用:FSMCaller的onCommitted(实际就是更新lastAppliedIndex).  