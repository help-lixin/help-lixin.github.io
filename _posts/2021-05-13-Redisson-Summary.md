---
layout: post
title: 'Redisson 简介(一)'
date: 2021-05-13
author: 李新
tags:  Redisson
---

### (1). Redisson是什么?
> Redisson是一个在Redis的基础上实现的Java驻内存数据网格(In-Memory Data Grid).  
> 它不仅提供了一系列的分布式的Java常用对象,还提供了许多分布式服务.其中包括(BitSet, Set, Multimap, SortedSet, Map, List, Queue, BlockingQueue, Deque, BlockingDeque, Semaphore, Lock, AtomicLong, CountDownLatch, Publish / Subscribe, Bloom filter, Remote service, Spring cache, Executor service, Live Object service, Scheduler service) Redisson提供了使用Redis的最简单和最便捷的方法.
> Redisson的宗旨是促进使用者对Redis的关注分离(Separation of Concern),从而让使用者能够将精力更集中地放在处理业务逻辑上.

### (2). Redisson 限流案例
```
package help.lixin.redisson;

import java.util.Date;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

import org.junit.Assert;
import org.junit.BeforeClass;
import org.junit.Test;
import org.redisson.Redisson;
import org.redisson.api.RAtomicLong;
import org.redisson.api.RKeys;
import org.redisson.api.RLock;
import org.redisson.api.RRateLimiter;
import org.redisson.api.RateIntervalUnit;
import org.redisson.api.RateType;
import org.redisson.api.RedissonClient;
import org.redisson.config.Config;
import org.redisson.config.TransportMode;

public class AllTest {

	private static RedissonClient client = null;

	@BeforeClass
	public static void init() {
		Config config = new Config();
		config.setTransportMode(TransportMode.NIO);
		// 监控狗,每隔10秒续租一次
		config.setLockWatchdogTimeout(10000);
		config.useSingleServer().setAddress("redis://127.0.0.1:6379");
		client = Redisson.create(config);
	} // end init
	
	// 测试限流
	@Test
	public void testRateLimiter() {
		RRateLimiter rateLimiter = client.getRateLimiter("myRateLimiter");
		// 初始化
		// 最大流速 = 每1秒钟产生2个令牌
		rateLimiter.trySetRate(RateType.OVERALL, 2, 1, RateIntervalUnit.SECONDS);

		// 创建三个线程来测试,通过日志上的时间来验证.
		Thread t1 = new Thread() {
			public void run() {
				System.out.println("t1 acquire start " + new Date());
				rateLimiter.acquire(1);
				System.out.println("t1 acquire end " + new Date());
			}
		};

		Thread t2 = new Thread() {
			public void run() {
				System.out.println("t2 acquire start " + new Date());
				rateLimiter.acquire(1);
				System.out.println("t2 acquire end " + new Date());
			}
		};

		Thread t3 = new Thread() {
			public void run() {
				System.out.println("t3 acquire start " + new Date());
				rateLimiter.acquire(1);
				System.out.println("t3 acquire end " + new Date());
			}
		};

		t1.start();
		t2.start();
		t3.start();

		CountDownLatch latch = new CountDownLatch(1);
		try {
			latch.await();
		} catch (InterruptedException e) {
		}
	} // end testRateLimiter
}
```

### (3). Redisson限流案例(结果分析)
```
# 令牌桶逻辑为:每1秒钟产生2个令牌
# t1,t2,t3同时进入,获取令牌方法.
t2 acquire start Thu May 13 17:13:41 CST 2021
t3 acquire start Thu May 13 17:13:41 CST 2021
t1 acquire start Thu May 13 17:13:41 CST 2021

# t1,t3是获取到了令牌(正好是1秒2个)
t1 acquire end Thu May 13 17:13:41 CST 2021
t3 acquire end Thu May 13 17:13:41 CST 2021

# t2 等待下一秒令牌的生成,并获取
t2 acquire end Thu May 13 17:13:42 CST 2021
```
### (4). Redisson分布式锁案例
```
package help.lixin.redisson;

import java.util.Date;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

import org.junit.Assert;
import org.junit.BeforeClass;
import org.junit.Test;
import org.redisson.Redisson;
import org.redisson.api.RAtomicLong;
import org.redisson.api.RKeys;
import org.redisson.api.RLock;
import org.redisson.api.RRateLimiter;
import org.redisson.api.RateIntervalUnit;
import org.redisson.api.RateType;
import org.redisson.api.RedissonClient;
import org.redisson.config.Config;
import org.redisson.config.TransportMode;

public class AllTest {

	private static RedissonClient client = null;

	@BeforeClass
	public static void init() {
		Config config = new Config();
		config.setTransportMode(TransportMode.NIO);
		// 监控狗,每隔10秒续租一次
		config.setLockWatchdogTimeout(10000);
		config.useSingleServer().setAddress("redis://127.0.0.1:6379");
		client = Redisson.create(config);
	} // end init
	
	@Test
	public void testLock() {
		CountDownLatch latch = new CountDownLatch(2);
		
		Runnable r = ()->{
			RLock lock = client.getLock("anyLock");
			try {
				System.out.println(Thread.currentThread().getName() + " lock start before " + new Date());
				// 只有在这种加锁(lock.lock())的模式下,看门狗才会帮你续租(访方法是个阻塞方法)
				lock.lock();

				// 以下这几种模式,都会自动释放锁,不会帮你进行续租
				// lock.lock(10, TimeUnit.SECONDS);
				// boolean res = lock.tryLock(100, 10, TimeUnit.SECONDS);
				System.out.println(Thread.currentThread().getName() + " lock SUCCESS,before Runntin Business " + new Date());
				// 模拟:业务执行时间为30秒,想知道看门狗是否会自动帮我续租.
				TimeUnit.SECONDS.sleep(30);
				System.out.println(Thread.currentThread().getName() + " lock SUCCESS,after Runntin Business " + new Date());
			} catch (InterruptedException ignore) {
			} finally {
				System.out.println(Thread.currentThread().getName() + " unlock " + new Date());
				lock.unlock();
			}
			latch.countDown();
		};
		
		Thread t1 = new Thread(r, "t1");
		Thread t2 = new Thread(r, "t2");
		t1.start();
		t2.start();
		
		try {
			latch.await();
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	} // end testLock
}
```
### (5). Redisson分布式锁案例(结果分析)
> 结论:lock.lock()方法是阻塞方法,同时,看门狗会帮忙续租,其余lock方法,锁会自动释放(结合Redis TTL验证).

``` 
# 在加锁之前:t1和t2同时,进入方法,并打印日志
t1 lock start before Thu May 13 17:04:00 CST 2021
t2 lock start before Thu May 13 17:04:00 CST 2021

# t2先加锁成功,执行业务逻辑(业务逻辑时间花费了30秒),然后,t2释放锁
t2 lock SUCCESS,before Runntin Business Thu May 13 17:04:00 CST 2021
t2 lock SUCCESS,after Runntin Business Thu May 13 17:04:30 CST 2021
t2 unlock Thu May 13 17:04:30 CST 2021

# 只有t2释放锁,t1才会继续执行.
t1 lock SUCCESS,before Runntin Business Thu May 13 17:04:30 CST 2021
t1 lock SUCCESS,after Runntin Business Thu May 13 17:05:00 CST 2021
t1 unlock Thu May 13 17:05:00 CST 2021
```

### (6). 总结
> 如果要使用令牌桶限流或者分布锁的话,直接使用:Redisson框架即可.    
> 后面,我也会着重对这两者的源码进行剖析.  
