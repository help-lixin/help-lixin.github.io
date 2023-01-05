---
layout: post
title: 'SOFAJRaft源码之FSMCaller下篇(十二)' 
date: 2022-12-28
author: 李新
tags:  SOFAJRaft
---

### (1). 概述
一直都好奇,FSMCaller与StateMachine的onApply方法是如何交互的,曾还一度怀疑可能不存在交互,这一小篇主要剖析:FSMCaller与StateMachine的onApply方法交互. 

### (2). FSMCallerTest
```
@RunWith(value = MockitoJUnitRunner.class)
public class FSMCallerTest {
    private static final String GROUP_ID = "group001";
    private FSMCallerImpl       fsmCaller;
    @Mock
    private NodeImpl            node;
    @Mock
    private StateMachine        fsm;
    @Mock
    private LogManager          logManager;
    private ClosureQueueImpl    closureQueue;

    @Before
    public void setup() {
        this.fsmCaller = new FSMCallerImpl();
        this.closureQueue = new ClosureQueueImpl(GROUP_ID);

        final FSMCallerOptions opts = new FSMCallerOptions();
        Mockito.when(this.node.getNodeMetrics()).thenReturn(new NodeMetrics(false));
        Mockito.when(this.node.getGroupId()).thenReturn(GROUP_ID);
        opts.setNode(this.node);
        opts.setFsm(this.fsm);
        opts.setLogManager(this.logManager);
        opts.setBootstrapId(new LogId(10, 1));
        opts.setClosureQueue(this.closureQueue);
        assertTrue(this.fsmCaller.init(opts));
    } // end setup

	
	@Test
    public void testOnCommitted() throws Exception {
        final LogEntry log = new LogEntry(EntryType.ENTRY_TYPE_DATA);
        log.getId().setIndex(11);
        log.getId().setTerm(1);
        Mockito.when(this.logManager.getTerm(11)).thenReturn(1L);
        Mockito.when(this.logManager.getEntry(11)).thenReturn(log);
        final ArgumentCaptor<Iterator> itArg = ArgumentCaptor.forClass(Iterator.class);

		// ******************************************************************
		// 提交committedIndex
		// ******************************************************************
        assertTrue(this.fsmCaller.onCommitted(11));
				
        this.fsmCaller.flush();
        Thread.sleep(30);
 	}	 // end testOnCommitted

}
```
### (3). FSMCaller.onCommitted
```
public boolean onCommitted(final long committedIndex) {
	// 在前面剖析过了,是扔给了:Disruptor
	return enqueueTask((task, sequence) -> {
		// 为Task配置Type和committedIndex,
		task.type = TaskType.COMMITTED;
		task.committedIndex = committedIndex;
	});
}
```
### (4). FSMCaller.runApplyTask
```
private long runApplyTask(final ApplyTask task, long maxCommittedIndex, final boolean endOfBatch) {
	CountDownLatch shutdown = null;
	if (task.type == TaskType.COMMITTED) {
		// maxCommittedIndex 为上一次提交的index,不过,每一次处理完后,都把:maxCommittedIndex还原成了-1来着的.
		if (task.committedIndex > maxCommittedIndex) {
			maxCommittedIndex = task.committedIndex;
		}
		task.reset();
	} else {
		// ... ...
	}
	
	
	try {
		if (endOfBatch && maxCommittedIndex >= 0) {
			this.currTask = TaskType.COMMITTED;
			// ***************************************************************************
			// 开始提交
			// ***************************************************************************
			doCommitted(maxCommittedIndex);
			// ***************************************************************************
			// 重置:maxCommittedIndex
			// ***************************************************************************
			maxCommittedIndex = -1L; // reset maxCommittedIndex
		}
		this.currTask = TaskType.IDLE;
		return maxCommittedIndex;
	} finally {
		if (shutdown != null) {
			shutdown.countDown();
		}
	} // end try
}	
```
### (5). FSMCaller.doCommitted
```
private void doCommitted(final long committedIndex) {
	if (!this.error.getStatus().isOk()) {
		return;
	}
	
	// 10 
	final long lastAppliedIndex = this.lastAppliedIndex.get();
	// We can tolerate the disorder of committed_index
	// 10 >= 11
	if (lastAppliedIndex >= committedIndex) { // false
		return;
	}
	
	// 把committedIndex改成:11
	this.lastCommittedIndex.set(committedIndex);
	
	final long startMs = Utils.monotonicMs();
	try {
		final List<Closure> closures = new ArrayList<>();
		final List<TaskClosure> taskClosures = new ArrayList<>();
		
		// 弹出数据
		final long firstClosureIndex = this.closureQueue.popClosureUntil(committedIndex, closures, taskClosures);

		// Calls TaskClosure#onCommitted if necessary
		onTaskCommitted(taskClosures);

		Requires.requireTrue(firstClosureIndex >= 0, "Invalid firstClosureIndex");
		final IteratorImpl iterImpl = new IteratorImpl(this, this.logManager, closures, firstClosureIndex, lastAppliedIndex, committedIndex, this.applyingIndex);
		while (iterImpl.isGood()) { // 游标并未移动来着的
			final LogEntry logEntry = iterImpl.entry();
			if (logEntry.getType() != EnumOutter.EntryType.ENTRY_TYPE_DATA) { // 如果日志内容类型不是数据
				if (logEntry.getType() == EnumOutter.EntryType.ENTRY_TYPE_CONFIGURATION) { // 如果日志类型为配置
					if (logEntry.getOldPeers() != null && !logEntry.getOldPeers().isEmpty()) {
						// Joint stage is not supposed to be noticeable by end users.
						// ******************************************************************************
						// 调用状态机的:onConfigurationCommitted
						// ******************************************************************************
						this.fsm.onConfigurationCommitted(new Configuration(iterImpl.entry().getPeers()));
					}
				}
				
				if (iterImpl.done() != null) {
					iterImpl.done().run(Status.OK());
				}
				iterImpl.next();
				continue;
			}
			
			// ******************************************************************************
			// 调用状态机apply任务
			// ******************************************************************************
			// Apply data task to user state machine
			doApplyTasks(iterImpl);
		}

		if (iterImpl.hasError()) {
			setError(iterImpl.getError());
			iterImpl.runTheRestClosureWithError();
		}
		
		// 12 - 1 = 11
		long lastIndex = iterImpl.getIndex() - 1;
		// 1 
		final long lastTerm = this.logManager.getTerm(lastIndex);
        
		// ******************************************************************************
		// 设置lastApplied
		// ******************************************************************************
		setLastApplied(lastIndex, lastTerm);
	} finally {
		this.nodeMetrics.recordLatency("fsm-commit", Utils.monotonicMs() - startMs);
	}
}
```
### (6). FSMCaller.doApplyTasks
```
private void doApplyTasks(final IteratorImpl iterImpl) {
	final IteratorWrapper iter = new IteratorWrapper(iterImpl);
	final long startApplyMs = Utils.monotonicMs();
	final long startIndex = iter.getIndex();
	try {
		// *******************************************************************************
		// 回调业务自定义的: StateMachine.onApply
		// *******************************************************************************
		this.fsm.onApply(iter);
	} finally {
		this.nodeMetrics.recordLatency("fsm-apply-tasks", Utils.monotonicMs() - startApplyMs);
		this.nodeMetrics.recordSize("fsm-apply-tasks-count", iter.getIndex() - startIndex);
	}
	
	// 如果迭代器里还有数据,则进行日志提示
	if (iter.hasNext()) {
		LOG.error("Iterator is still valid, did you return before iterator reached the end?");
	}
	
	// 尝试移动到下一个日志(这里的next是被JRAFT给重写过了的)
	// Try move to next in case that we pass the same log twice.
	iter.next();
}
```
### (7). FSMCaller.setLastApplied
```
void setLastApplied(long lastIndex, final long lastTerm) {
	// lastIndex = 11
	// lastTerm = 1
	final LogId lastAppliedId = new LogId(lastIndex, lastTerm);
	this.lastAppliedIndex.set(lastIndex);
	this.lastAppliedTerm = lastTerm;
	this.logManager.setAppliedId(lastAppliedId);
	// 回调:LastAppliedLogIndexListener
	notifyLastAppliedIndexUpdated(lastIndex);
}
```
### (8). 总结
> FSMCaller与StateMachine是什么关系呢?   
> FSMCaller是一个中控管理类,可以理解,StateMachine是FSMCaller留出来的一个接口(钩子),只是,这个接口,要求开发员自己必须要实现而已.   