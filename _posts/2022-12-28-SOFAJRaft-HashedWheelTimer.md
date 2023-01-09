---
layout: post
title: 'SOFAJRaft源码之HashedWheelTimer(二十)' 
date: 2022-12-28
author: 李新
tags:  SOFAJRaft
---

### (1). 概述

### (2). HashedWheelTimerTest
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
	mask = wheel.length - 1;

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
### (6). 

### (7). 

### (8). 

### (9). 

### (10). 
 
### (11). 

### (12). 