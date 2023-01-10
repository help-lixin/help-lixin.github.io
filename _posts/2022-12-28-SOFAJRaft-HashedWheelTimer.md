---
layout: post
title: 'SOFAJRaft源码之HashedWheelTimer(二十)' 
date: 2022-12-28
author: 李新
tags:  SOFAJRaft
---

### (1). 概述
前面剖析了与网络相关的内容,按理来说:应该要开始剖析Replicator,但却发现,如果用Mock来走测试案例剖析的话,自我感觉太虚了,
所以,换种方式来解读源码,解析一套网络流程,在剖析之前,要先把Node里所依赖的一些对象给先剖析掉,以免受影响,所以,这一篇主要剖析
HashedWheelTimer,没有想到的是:JRAFT是直接拷贝Netty的代码.

### (2). HashedWheelTimer简单使用
```
public class HashedWheelTimerTest {

    @Test
    public void testHashedWheelTimer() throws InterruptedException {
        HashedWheelTimer hashedWheelTimer = new HashedWheelTimer(1, TimeUnit.SECONDS, 12);
        hashedWheelTimer.start();

        Timeout timeout = hashedWheelTimer.newTimeout((t) -> {
            System.out.println("--------------------->" + t);
        }, 10, TimeUnit.SECONDS);

        TimeUnit.SECONDS.sleep(30);

        // timeout.cancel();

        hashedWheelTimer.stop();
    }
}
```
### (3). HashedWheelTimer初始化
```
public HashedWheelTimer(
                 ThreadFactory threadFactory, 
				 // 每一格之间的时间间隔
				 // 1
				 long tickDuration, 
				 // TimeUnit.SECONDS
				 TimeUnit unit, 
				 // 12 
				 int ticksPerWheel,
				 long maxPendingTimeouts) {
	if (threadFactory == null) {
		throw new NullPointerException("threadFactory");
	}
	if (unit == null) {
		throw new NullPointerException("unit");
	}
	if (tickDuration <= 0) {
		throw new IllegalArgumentException("tickDuration must be greater than 0: " + tickDuration);
	}
	if (ticksPerWheel <= 0) {
		throw new IllegalArgumentException("ticksPerWheel must be greater than 0: " + ticksPerWheel);
	}

    // **************************************************************************
	// 1. 创建,并初始化:HashedWheelBucket[]数组
	// **************************************************************************
	// Normalize ticksPerWheel to power of two and initialize the wheel.
	wheel = createWheel(ticksPerWheel);
	// 在后面,会拿着tick和mask进行与计算,最终的值一定是在mask范围之内. 
	mask = wheel.length - 1;


	// **************************************************************************
	// 把时间转换成纳秒: 1秒 = 1000000000纳秒
	// **************************************************************************
	// Convert tickDuration to nanos.
	this.tickDuration = unit.toNanos(tickDuration);

	// Prevent overflow.
	if (this.tickDuration >= Long.MAX_VALUE / wheel.length) {
		throw new IllegalArgumentException(String.format(
			"tickDuration: %d (expected: 0 < tickDuration in nanos < %d", tickDuration, Long.MAX_VALUE
																						/ wheel.length));
	}
	
	// *******************************************************
	// 2. 通过工厂,创建Work线程
	// *******************************************************
	workerThread = threadFactory.newThread(worker);

	this.maxPendingTimeouts = maxPendingTimeouts;

	if (instanceCounter.incrementAndGet() > INSTANCE_COUNT_LIMIT
		&& warnedTooManyInstances.compareAndSet(false, true)) {
		reportTooManyInstances();
	}
} // end HashedWheelTimer构造器

private static HashedWheelBucket[] createWheel(int ticksPerWheel) {
	// 对进行校验,控制在2的30幂
	if (ticksPerWheel <= 0) {
		throw new IllegalArgumentException("ticksPerWheel must be greater than 0: " + ticksPerWheel);
	}
	if (ticksPerWheel > 1073741824) {
		throw new IllegalArgumentException("ticksPerWheel may not be greater than 2^30: " + ticksPerWheel);
	}
	
	// 
	ticksPerWheel = normalizeTicksPerWheel(ticksPerWheel);
	HashedWheelBucket[] wheel = new HashedWheelBucket[ticksPerWheel];
	for (int i = 0; i < wheel.length; i++) {
		wheel[i] = new HashedWheelBucket();
	}
	return wheel;
} // end createWheel

private static int normalizeTicksPerWheel(int ticksPerWheel) {
	int normalizedTicksPerWheel = 1;
	while (normalizedTicksPerWheel < ticksPerWheel) {
		normalizedTicksPerWheel <<= 1;
	}
	return normalizedTicksPerWheel;
} // end normalizeTicksPerWheel
```
### (4). HashedWheelTimer.newTimeout
> HashedWheelTimer添加任务. 

```
// timeouts
private final Queue<HashedWheelTimeout>  timeouts = new ConcurrentLinkedQueue<>();

public Timeout newTimeout(TimerTask task, long delay, TimeUnit unit) {
	// ... ...

   // 启动Work线程
	start();

	// ... ...
	
	// deadline 
	// 执行时间为: 当前时间(纳秒) + 延迟时间 - 启动时间
	// 所以,最终执行时间为: 逻辑时间哈.
	long deadline = System.nanoTime() + unit.toNanos(delay) - startTime;
	
	// **************************************************************************
	// 把task转换成:HashedWheelTimeout,添加到队列里(timeouts)
	// **************************************************************************
	HashedWheelTimeout timeout = new HashedWheelTimeout(this, task, deadline);
	timeouts.add(timeout);
	return timeout;
}// end newTimeout


// HashedWheelTimer属性部份
// workerStateUpdater
private static final AtomicIntegerFieldUpdater<HashedWheelTimer> workerStateUpdater  = 
                                AtomicIntegerFieldUpdater.newUpdater(HashedWheelTimer.class,"workerState");
// work状态								
private volatile int                                             workerState;
// work初始化
public static final int                                          WORKER_STATE_INIT      = 0;
// work启动
public static final int                                          WORKER_STATE_STARTED   = 1;
// work关闭
public static final int                                          WORKER_STATE_SHUTDOWN  = 2;
// 启动初始化latch
private final CountDownLatch                                     startTimeInitialized   = new CountDownLatch(1);

public void start() {
	switch (workerStateUpdater.get(this)) {
		case WORKER_STATE_INIT:
		    // **************************************************************************
			// 实际比较简单,通过元子性验证workerState的状态是初始化时,设置成:启动中,并且,启动work线程
			// **************************************************************************
			if (workerStateUpdater.compareAndSet(this, WORKER_STATE_INIT, WORKER_STATE_STARTED)) {
				workerThread.start();
			}
			break;
		case WORKER_STATE_STARTED:
			break;
		case WORKER_STATE_SHUTDOWN:
			throw new IllegalStateException("cannot be started once stopped");
		default:
			throw new Error("Invalid WorkerState");
	}

	// Wait until the startTime is initialized by the worker.
	while (startTime == 0) {
		try {
			
			startTimeInitialized.await();
		} catch (InterruptedException ignore) {
		}
	}
} // end start
```
### (5). Timeout.cancel
> 添加任务之后,会返回:Timeout,我们可以Hold住这个对象,进行任务的取消. 

```

// 取消任务列表
private final Queue<HashedWheelTimeout> cancelledTimeouts = new ConcurrentLinkedQueue<>();

// 取消任务
public boolean cancel() {
	// 比较状态,
	if (!compareAndSetState(ST_INIT, ST_CANCELLED)) {
		return false;
	}
	
	// 把要取消的任务,添加到队列里.
	timer.cancelledTimeouts.add(this);
	return true;
}
```
### (6). Work.run
```
private final class Worker implements Runnable {
	private final Set<Timeout> unprocessedTimeouts = new HashSet<>();

	private long               tick;

	@Override
	public void run() {
		
		// 当前的纳秒数
		// Initialize the startTime.
		startTime = System.nanoTime();
		if (startTime == 0) {
			// We use 0 as an indicator for the uninitialized value here, so make sure it's not 0 when initialized.
			startTime = 1;
		}

		// Notify the other threads waiting for the initialization at start().
		// 配合前面的添加任务.
		startTimeInitialized.countDown();

		do {
			// ************************************************************************
			// 1. 计算出下一轮的时间
			// ************************************************************************
			final long deadline = waitForNextTick();
			if (deadline > 0) {
				// ***************************************************************
				// 2. 这里与运算,所以,最终的idx是:0~511之间,这样,就实际对应的着数组512以内.
				// ***************************************************************
				int idx = (int) (tick & mask);
				
				// ***************************************************************
				// 3. 处理取消的任务
				// ***************************************************************
				processCancelledTasks();
				
				// ***************************************************************
				// 4. 根据idx从下标中取出来,与第2步是对应的.
				// ***************************************************************
				HashedWheelBucket bucket = wheel[idx];
				
				// ***************************************************************
				// 5. 把队列中的任务,移到桶里.
				// ***************************************************************
				transferTimeoutsToBuckets();
				bucket.expireTimeouts(deadline);
				tick++;
			}
		} while (workerStateUpdater.get(HashedWheelTimer.this) == WORKER_STATE_STARTED);

		// Fill the unprocessedTimeouts so we can return them from stop() method.
		for (HashedWheelBucket bucket : wheel) {
			bucket.clearTimeouts(unprocessedTimeouts);
		}
		
		
		for (;;) {
			HashedWheelTimeout timeout = timeouts.poll();
			if (timeout == null) {
				break;
			}
			if (!timeout.isCancelled()) {
				unprocessedTimeouts.add(timeout);
			}
		}
		
		processCancelledTasks();
	}// end run
}		
```
### (7). Work.waitForNextTick
```
private long waitForNextTick() {
	// tick自增
	// tickDuration: 1秒 = 1000000000纳秒
	// 这一步的做法是什么呢? 实际就是下一秒哈
	long deadline = tickDuration * (tick + 1);

	for (;;) {
		final long currentTime = System.nanoTime() - startTime;
		long sleepTimeMs = (deadline - currentTime + 999999) / 1000000;

		if (sleepTimeMs <= 0) {
			if (currentTime == Long.MIN_VALUE) {
				return -Long.MAX_VALUE;
			} else {
				return currentTime;
			}
		}
		
		try {
			Thread.sleep(sleepTimeMs);
		} catch (InterruptedException ignored) {
			if (workerStateUpdater.get(HashedWheelTimer.this) == WORKER_STATE_SHUTDOWN) {
				return Long.MIN_VALUE;
			}
		}
	}
}
```
### (8). Work.transferTimeoutsToBuckets
```
private void transferTimeoutsToBuckets() {
	// 额,最多拿10W个任务处理
	// transfer only max. 100000 timeouts per tick to prevent a thread to stale the workerThread when it just
	// adds new timeouts in a loop.
	for (int i = 0; i < 100000; i++) {
		HashedWheelTimeout timeout = timeouts.poll();
		// 没有任务
		if (timeout == null) {
			// all processed
			break;
		}
		
		// 任务被取消掉了
		if (timeout.state() == HashedWheelTimeout.ST_CANCELLED) {
			// Was cancelled in the meantime.
			continue;
		}

		
		long calculated = timeout.deadline / tickDuration;
		timeout.remainingRounds = (calculated - tick) / wheel.length;

		final long ticks = Math.max(calculated, tick); // Ensure we don't schedule for past.
		int stopIndex = (int) (ticks & mask);

		// ***********************************************************************************  
		// 找到对应的桶,添加timeout
		// ***********************************************************************************  
		HashedWheelBucket bucket = wheel[stopIndex];
		bucket.addTimeout(timeout);
	}
}
```

### (9). Work.expireTimeouts

```
public void expireTimeouts(long deadline) {
	HashedWheelTimeout timeout = head;

	// process all timeouts
	while (timeout != null) {
		HashedWheelTimeout next = timeout.next;
		if (timeout.remainingRounds <= 0) {
			next = remove(timeout);
			if (timeout.deadline <= deadline) {
				// ******************************************************
				// 执行TimerTask.run方法
				// ******************************************************
				timeout.expire();
			} else {
				// The timeout was placed into a wrong slot. This should never happen.
				throw new IllegalStateException(String.format("timeout.deadline (%d) > deadline (%d)",
					timeout.deadline, deadline));
			}
		} else if (timeout.isCancelled()) {
			next = remove(timeout);
		} else {
			timeout.remainingRounds--;
		}
		timeout = next;
	}
}
```
### (10). 总结
HashedWheelTimer的使用还是有限的,是以HashedWheelTimer启动为基准,一轮一轮的走下去来着的,而且,添加任务时,是扔到队列里,Work线程会一直走下去,取出队列里的数据,复制到桶里,并遍历桶(链表)执行:TimerTask. 