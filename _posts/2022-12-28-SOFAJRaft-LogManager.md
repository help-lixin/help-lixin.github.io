---
layout: post
title: 'SOFAJRaft源码之LogManager常用方法剖析(五)' 
date: 2022-12-28
author: 李新
tags:  SOFAJRaft
---

### (1). 概述
还是按照官网的框架图,以局部作为点,进行剖析,这一小篇主要分析:LogManager,它是对LogStorage进行了包装,为什么要包装呢?因为,这个类它在LogStorage的基础上增加了:缓存和批量提交的功能,咱们一点点的剖析. 
### (2). LogManager追加日志(LogManager.appendEntries)
```
public void appendEntries(final List<LogEntry> entries, final StableClosure done) {
	assert(done != null);

	Requires.requireNonNull(done, "done");
	if (this.hasError) { // 如果追加日志有错的情况下处理
		entries.clear();
		ThreadPoolsFactory.runClosureInThread(this.groupId, done, new Status(RaftError.EIO, "Corrupted LogStorage"));
		return;
	}

	boolean doUnlock = true;
	this.writeLock.lock();
	try {
		// 检查要追加的日志
		// 如果追加的日志的第一个日志的index是0的情况下,对日志的index进行重新编排,使其日志index是从1开始
		// lastLogIndex
		if (!entries.isEmpty() && !checkAndResolveConflict(entries, done, this.writeLock)) {
			// If checkAndResolveConflict returns false, the done will be called in it.
			entries.clear();
			return;
		}

		// 遍历日志,当日志类型为:CONFIGURATION,对其进行特殊处理
		for (int i = 0; i < entries.size(); i++) {
			final LogEntry entry = entries.get(i);
			// Set checksum after checkAndResolveConflict
			if (this.raftOptions.isEnableLogEntryChecksum()) {
				entry.setChecksum(entry.checksum());
			}
			if (entry.getType() == EntryType.ENTRY_TYPE_CONFIGURATION) {
				Configuration oldConf = new Configuration();
				if (entry.getOldPeers() != null) {
					oldConf = new Configuration(entry.getOldPeers(), entry.getOldLearners());
				}
				final ConfigurationEntry conf = new ConfigurationEntry(entry.getId(),
					new Configuration(entry.getPeers(), entry.getLearners()), oldConf);
				this.configManager.add(conf);
			}
		}

		// 日志集合不为空的情况下
		if (!entries.isEmpty()) {
			// 为钩子函数配置第一个日志为集合中的第一个元数
			done.setFirstLogIndex(entries.get(0).getId().getIndex());
			// *****************************************************************************
			// 把日志集合,添加到内存集合里进行缓存,等处理完后,再从缓存中取出.
			// *****************************************************************************
			this.logsInMemory.addAll(entries);
		}
		// *****************************************************************************
		// 为钩子函数配置日志集合
		// *****************************************************************************
		done.setEntries(entries);

		doUnlock = false;
		// 这一段可以理解为:遍历所有的:LastLogIndexListener,挨个调用:onLastLogIndexChanged方法
		if (!wakeupAllWaiter(this.writeLock)) {
			notifyLastLogIndexListeners();
		}

		// *****************************************************************************
		// 可以这样直接向Disruptor直接发布事件了,不需要先拿sequencer了
		// 看了下publishEvent代码,内部实际帮你拿了sequencer进行发布.
		// *****************************************************************************
		// publish event out of lock
		this.diskQueue.publishEvent((event, sequence) -> {
		  event.reset();
		  event.type = EventType.OTHER;
		  event.done = done;
		});
	} finally {
		if (doUnlock) {
			this.writeLock.unlock();
		}
	}
}
```
### (3). StableClosureEventHandler.onEvent
```
private class StableClosureEventHandler implements EventHandler<StableClosureEvent> {
	LogId               lastId  = LogManagerImpl.this.diskId;
	List<StableClosure> storage = new ArrayList<>(256);
	// ********************************************************************************************
	// StableClosureEventHandler是单例的哈
	// AppendBatcher一看类名就知道是批量追加数据
	// ********************************************************************************************
	AppendBatcher       ab      = new AppendBatcher(this.storage, 256, new ArrayList<>(), LogManagerImpl.this.diskId);

	@Override
	public void onEvent(final StableClosureEvent event, final long sequence, final boolean endOfBatch)
																									  throws Exception {
		if (event.type == EventType.SHUTDOWN) { // 针对关闭的处理
			this.lastId = this.ab.flush();
			setDiskId(this.lastId);
			LogManagerImpl.this.shutDownLatch.countDown();
			event.reset();
			return;
		}
		final StableClosure done = event.done;
		final EventType eventType = event.type;

		event.reset();

		if (done.getEntries() != null && !done.getEntries().isEmpty()) {
			// **************************************************************
			// 1. 委派给:AppendBatcher.append方法追加数据(数据被包在done里)
			// **************************************************************
			this.ab.append(done);
		} else {
			this.lastId = this.ab.flush();
			boolean ret = true;
			switch (eventType) {
				case LAST_LOG_ID:
					((LastLogIdClosure) done).setLastLogId(this.lastId.copy());
					break;
				case TRUNCATE_PREFIX:
					long startMs = Utils.monotonicMs();
					try {
						final TruncatePrefixClosure tpc = (TruncatePrefixClosure) done;
						LOG.debug("Truncating storage to firstIndexKept={}.", tpc.firstIndexKept);
						ret = LogManagerImpl.this.logStorage.truncatePrefix(tpc.firstIndexKept);
					} finally {
						LogManagerImpl.this.nodeMetrics.recordLatency("truncate-log-prefix", Utils.monotonicMs()
																							 - startMs);
					}
					break;
				case TRUNCATE_SUFFIX:
					startMs = Utils.monotonicMs();
					try {
						final TruncateSuffixClosure tsc = (TruncateSuffixClosure) done;
						LOG.warn("Truncating storage to lastIndexKept={}.", tsc.lastIndexKept);
						ret = LogManagerImpl.this.logStorage.truncateSuffix(tsc.lastIndexKept);
						if (ret) {
							this.lastId.setIndex(tsc.lastIndexKept);
							this.lastId.setTerm(tsc.lastTermKept);
							Requires.requireTrue(this.lastId.getIndex() == 0 || this.lastId.getTerm() != 0);
						}
					} finally {
						LogManagerImpl.this.nodeMetrics.recordLatency("truncate-log-suffix", Utils.monotonicMs()
																							 - startMs);
					}
					break;
				case RESET:
					final ResetClosure rc = (ResetClosure) done;
					LOG.info("Resetting storage to nextLogIndex={}.", rc.nextLogIndex);
					ret = LogManagerImpl.this.logStorage.reset(rc.nextLogIndex);
					break;
				default:
					break;
			}

			if (!ret) {
				reportError(RaftError.EIO.getNumber(), "Failed operation in LogStorage");
			} else {
				done.run(Status.OK());
			}
		}
		if (endOfBatch) {
			// *********************************************************************
			// 2. 刷新缓存
			// *********************************************************************
			this.lastId = this.ab.flush();
			setDiskId(this.lastId);
		}
	}

}
```
### (4). AppendBatcher.append
```
void append(final StableClosure done) {
	if (this.size == this.cap || this.bufferSize >= LogManagerImpl.this.raftOptions.getMaxAppendBufferSize()) {
		flush();
	}
	this.storage.add(done);
	this.size++;
	// *****************************************************************
	// 仅仅是把日志添加到待追加的集合中
	// *****************************************************************
	this.toAppend.addAll(done.getEntries());
	for (final LogEntry entry : done.getEntries()) {
		this.bufferSize += entry.getData() != null ? entry.getData().remaining() : 0;
	}
} // end 
```
### (5). AppendBatcher.flush
```
LogId flush() {
	if (this.size > 0) {
		// *******************************************************************
		// 1. 委派给:LogManager.appendToStorage进行日志的追加
		// *******************************************************************
		this.lastId = appendToStorage(this.toAppend);
		for (int i = 0; i < this.size; i++) {
			this.storage.get(i).getEntries().clear();
			Status st = null;
			try {
				if (LogManagerImpl.this.hasError) {
					st = new Status(RaftError.EIO, "Corrupted LogStorage");
				} else {
					st = Status.OK();
				}
				// *****************************************************************
				// 2. 回调钩子函数.
				// *****************************************************************
				this.storage.get(i).run(st);
			} catch (Throwable t) {
				LOG.error("Fail to run closure with status: {}.", st, t);
			}
		}
		this.toAppend.clear();
		this.storage.clear();

	}
	this.size = 0;
	this.bufferSize = 0;
	return this.lastId;
} // end 
```
### (6). LogManager.appendToStorage
```
private LogId appendToStorage(final List<LogEntry> toAppend) {
	LogId lastId = null;
	if (!this.hasError) {
		final long startMs = Utils.monotonicMs();
		final int entriesCount = toAppend.size();
		this.nodeMetrics.recordSize("append-logs-count", entriesCount);
		try {
			int writtenSize = 0;
			for (int i = 0; i < entriesCount; i++) {
				final LogEntry entry = toAppend.get(i);
				writtenSize += entry.getData() != null ? entry.getData().remaining() : 0;
			}
			this.nodeMetrics.recordSize("append-logs-bytes", writtenSize);
			// **************************************************************************
			// 委派给:LogStorage追加数据
			// **************************************************************************
			final int nAppent = this.logStorage.appendEntries(toAppend);
			if (nAppent != entriesCount) {
				LOG.error("**Critical error**, fail to appendEntries, nAppent={}, toAppend={}", nAppent,
					toAppend.size());
				reportError(RaftError.EIO.getNumber(), "Fail to append log entries");
			}
			if (nAppent > 0) {
				lastId = toAppend.get(nAppent - 1).getId();
			}
			toAppend.clear();
		} finally {
			this.nodeMetrics.recordLatency("append-logs", Utils.monotonicMs() - startMs);
		}
	}
	return lastId;
}
```
### (7). 总结
实际上,LogManager在追加日志时,是利用Disruptor异步线程去做的,并且,在追加日志可以以批量为单位,在追加之前先进行缓存,当追加日志完毕后,会再清理下缓存. 