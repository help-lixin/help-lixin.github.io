---
layout: post
title: 'SOFAJRaft源码之ClosureQueue(十三)' 
date: 2022-12-28
author: 李新
tags:  SOFAJRaft
---

### (1). 概述
本来应该要剖析StateMachine的,但是,在剖析FSMCaller(onCommitted方法)时,有涉及到一个接口:ClosureQueue,所以,在这一篇主要剖析它了.
### (2). 先看下ClosureQueue接口签名
> 有点懒了,就不画UML图了,因为,这个接口比较简单.

```
public interface ClosureQueue {
	
	// 清除所有的闭包
    void clear();

    // 重置第一个索引
    void resetFirstIndex(final long firstIndex);

    // 添加闭包到队列中
    void appendPendingClosure(final Closure closure);

    // 遍历队列,从列首开始弹出endIndex个数据
	// 如果数据类型是:TaskClosure,则,把数据添加到集合taskClosures里. 
	// 其余所有的Closure,则,把数据添加到集合closures里.
    long popClosureUntil(final long endIndex, final List<Closure> closures, final List<TaskClosure> taskClosures);
}
```
### (3). 看下ClosureQueue的默认实现类
> 能从ClosureQueue的实现类里看出来,它应该是在LinkedList的基础上做了一层业务逻辑的包装. 

```
public class ClosureQueueImpl implements ClosureQueue {

    private static final Logger LOG = LoggerFactory.getLogger(ClosureQueueImpl.class);

    private String              groupId;
    private final Lock          lock;
    private long                firstIndex;
	
	// *******************************************************************
	// 线性链表
	// *******************************************************************
    private LinkedList<Closure> queue;

    @OnlyForTest
    public long getFirstIndex() {
        return firstIndex;
    }

    @OnlyForTest
    public LinkedList<Closure> getQueue() {
        return queue;
    }

    public ClosureQueueImpl() {
        super();
        this.lock = new ReentrantLock();
        this.firstIndex = 0;
        this.queue = new LinkedList<>();
    }
	// ... ...
}	
```
### (4). ClosureQueueTest
```
public class ClosureQueueTest {
    private static final String GROUP_ID = "group001";
    private ClosureQueueImpl    queue;

    @Before
    public void setup() {
        this.queue = new ClosureQueueImpl(GROUP_ID);
    } // end setup

    @SuppressWarnings("SameParameterValue")
    private Closure mockClosure(final CountDownLatch latch) {
        return status -> {
            if (latch != null) {
                latch.countDown();
            }
        };
    } // end mockClosure

    @Test
    public void testAppendPop() {
        for (int i = 0; i < 10; i++) {
			// *******************************************************************
			// 添加闭包到队列中
			// *******************************************************************
            this.queue.appendPendingClosure(mockClosure(null));
        }
		
		
        List<Closure> closures = new ArrayList<>();
        assertEquals(0, this.queue.popClosureUntil(4, closures));
        assertEquals(5, closures.size());
	}  // end testAppendPop
}		
```
### (5). ClosureQueue.appendPendingClosure
> appendPendingClosure很简单,把数据,添加到队列的列尾. 

```
// private LinkedList<Closure> queue;

public void appendPendingClosure(final Closure closure) {
	this.lock.lock();
	try {
		// 添加闭包到队列的队尾
		this.queue.add(closure);
	} finally {
		this.lock.unlock();
	}
}
```
### (6). ClosureQueue.popClosureUntil
```
// closures     存储 Closure的结果集
// taskClosures 存储 TaskClosure的结果集
public long popClosureUntil(final long endIndex, final List<Closure> closures, final List<TaskClosure> taskClosures) {
	
	// 清空结果集. 
	closures.clear();
	if (taskClosures != null) {
		taskClosures.clear();
	}

	this.lock.lock();
	try {
		// 队列里的数据数量.
		final int queueSize = this.queue.size();
		// 如果队列里没有数据,又或者,endIndex 又小于 firstIndex,则返回:endIndex+1,不太理解这逻辑,先不管了.
		// 我的理解是:无论如何,你要取的数据得存在吧,这个返回有什么意义? 
		if (queueSize == 0 || endIndex < this.firstIndex) {
			return endIndex + 1;
		}

         // 这一步是验证: 
		 // endIndex : 要取的数据数量
		 // this.firstIndex + queueSize - 1 : 为队列里的实际数量
		 // 如果,要取的数据量大于队列里的数量,肯定是不能拿出那么多数据的. 
		if (endIndex > this.firstIndex + queueSize - 1) {
			LOG.error("Invalid endIndex={}, firstIndex={}, closureQueueSize={}", endIndex, this.firstIndex, queueSize);
			return -1;
		}

		// ****************************************************************************
		// firstIndex我咋感觉是一个计数器呢? 
        // 从队列里弹出endIndex + 1个元数
		// ****************************************************************************
		final long outFirstIndex = this.firstIndex;
		for (long i = outFirstIndex; i <= endIndex; i++) {
			// ****************************************************************************
			// 从队列的队首弹出一个元素
			// ****************************************************************************
			final Closure closure = this.queue.pollFirst();
			
			// 如果是:TaskClosure添加,把它添加到集合:taskClosures里
			if (taskClosures != null && closure instanceof TaskClosure) {
				taskClosures.add((TaskClosure) closure);
			}
			
			// 这里不是排除关系哦,这个closures可以包含:Closure和TaskClosure
			closures.add(closure);
		}

		this.firstIndex = endIndex + 1;
		return outFirstIndex;
	} finally {
		this.lock.unlock();
	}
}
```
### (7). 总结
ClosureQueue在LinkedList的基础上,提供了添加到队列和弹出队列的功能. 