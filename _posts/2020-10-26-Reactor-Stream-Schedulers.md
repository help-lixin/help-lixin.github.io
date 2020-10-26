---
layout: post
title: 'Reactor Stream源码(Schedulers)'
date: 2020-10-26
author: 李新
tags: ProjectReactorStream
---

### (1).Schedulers
```
//1. 创建线程
Schedulers.newElastic("selfTrade-thread")
```
### (2).Schedulers.newElastic
```
// 1. 
public static Scheduler newElastic(String name) {
    // name = "selfTrade-thread"
    return newElastic(name, ElasticScheduler.DEFAULT_TTL_SECONDS);
  }
  
  // 2.
  public static Scheduler newElastic(String name, int ttlSeconds) {
    // name = "selfTrade-thread"
    // ttlSeconds = 60 
    return newElastic(name, ttlSeconds, false);
  }
  
  // 3.
  public static Scheduler newElastic(String name, int ttlSeconds) {
      // name = "selfTrade-thread"
      // ttlSeconds = 60 
      return newElastic(name, ttlSeconds, false);
  }
  
  // 4. 
  public static Scheduler newElastic(String name, int ttlSeconds, boolean daemon) 
      return newElastic(
         ttlSeconds,
         // 4.1 
         new ReactorThreadFactory(     
               // selfTrade-thread
                name, 
                // 统计信息
                // AtomicLong COUNTER = new AtomicLong();
                ElasticScheduler.COUNTER, 
                // false
                daemon, 
                // 
                false,
                // 异常处理方法
		     Schedulers::defaultUncaughtException)
         );
	}
 
 // 5 
 // 
 static volatile Factory factory = DEFAULT;
 static final Factory DEFAULT = new Factory() { };
 public static Scheduler newElastic(int ttlSeconds, ThreadFactory threadFactory) {
    // ttlSeconds = 60
    // threadFactory = reactor.core.scheduler.ReactorThreadFactory
    // fatory 指向了自己内部的一个属性:DEFAULT
    // factory = Schedulers.DEFAULT
    return factory.newElastic(ttlSeconds, threadFactory);
 }
 
 
 
 public interface Schedulers.Factory {
     // 6. 
     default Scheduler newElastic(int ttlSeconds, ThreadFactory threadFactory) {
       // ttlSeconds = 60 
       // threadFactory = reactor.core.scheduler.ReactorThreadFactory
       // 创建ElasticScheduler并返回(看第5点)
       return new ElasticScheduler(threadFactory, ttlSeconds);
	 }
  
 }
```
### (3).ReactorThreadFactory
```
package reactor.core.scheduler;

class ReactorThreadFactory 
       implements ThreadFactory,
                  Supplier<String>,
                  Thread.UncaughtExceptionHandler {
    // 线程名称
    final private String                        name;
	// 统计信息
   final private AtomicLong                    counterReference;
   // 是否守护线程 false
	final private boolean                       daemon;
   // 是否阻塞并拒绝 false
	final private boolean                       rejectBlocking;

    @Nullable
	final private BiConsumer<Thread, Throwable> uncaughtExceptionHandler;
   
	ReactorThreadFactory(String name,
			AtomicLong counterReference,
			boolean daemon,
			boolean rejectBlocking,
			@Nullable BiConsumer<Thread, Throwable> uncaughtExceptionHandler) {
		this.name = name;
		this.counterReference = counterReference;
		this.daemon = daemon;
		this.rejectBlocking = rejectBlocking;
            // 异常处理方式
		this.uncaughtExceptionHandler = uncaughtExceptionHandler;
	}
 
 
 @Override
	public final Thread newThread(@NotNull Runnable runnable) {
           // 创建线程名称
		String newThreadName = name + "-" + counterReference.incrementAndGet();
		// rejectBlocking  = false
           Thread t = rejectBlocking
                      ? new NonBlockingThread(runnable, newThreadName)
                      : new Thread(runnable, newThreadName);
		if (daemon) {
              t.setDaemon(true);
		}
		if (uncaughtExceptionHandler != null) {
              t.setUncaughtExceptionHandler(this);
		}
		return t;
	}

   // 拒绝策略
	@Override
	public void uncaughtException(Thread t, Throwable e) {
        if (uncaughtExceptionHandler == null) {
            return;
        }
        uncaughtExceptionHandler.accept(t,e);
	}
    
    // 获得线程名称
	@Override
	public final String get() {
        return name;
	}

    static final class NonBlockingThread extends Thread implements NonBlocking {
        public NonBlockingThread(Runnable target, String name) {
            super(target, name);
         }
    }// end NonBlockingThread

}
```
### (4).Scheduler
```
public interface Scheduler extends Disposable {
    
    Disposable schedule(Runnable task);
    
    default long now(TimeUnit unit) {
        return unit.convert(System.currentTimeMillis(), TimeUnit.MILLISECONDS);
	}
 
     Worker createWorker();
     
    default void dispose() {
	 }
     
    default void start() {
	 }
    
     interface Worker extends Disposable {
         Disposable schedule(Runnable task);
         default Disposable schedule(Runnable task, long delay, TimeUnit unit) {
             throw Exceptions.failWithRejectedNotTimeCapable();
	      }
       
          default Disposable schedulePeriodically(Runnable task, long initialDelay, long period, TimeUnit unit) {
                throw Exceptions.failWithRejectedNotTimeCapable();
	       }
     } //end Worker    
    
}
```
### (5).ElasticScheduler
```
final class ElasticScheduler 
      // Reactor自定义的线程调度方法
      implements Scheduler, 
      // 返回JDK自带的线程池
      Supplier<ScheduledExecutorService>,
      //  
      Scannable {
    // 每创建一个线程 COUNTER++
    static final AtomicLong COUNTER = new AtomicLong();
    
    // 创建回收线程,并设置为守护线程 
    static final ThreadFactory EVICTOR_FACTORY = r -> {
		Thread t = new Thread(r, "elastic-evictor-" + COUNTER.incrementAndGet());
		t.setDaemon(true);
		return t;
	};
     
   static final CachedService SHUTDOWN = new CachedService(null);
	static final int DEFAULT_TTL_SECONDS = 60;
	final ThreadFactory factory;
	final int ttlSeconds;
	final Deque<ScheduledExecutorServiceExpiry> cache;
	final Queue<CachedService> all;
	final ScheduledExecutorService evictor;
	volatile boolean shutdown;
    
    // 1. 
    ElasticScheduler(ThreadFactory factory, int ttlSeconds) {
           // ttlSeconds = 60 
		if (ttlSeconds < 0) {
			throw new IllegalArgumentException("ttlSeconds must be positive, was: " + ttlSeconds);
		}
		this.ttlSeconds = ttlSeconds;
           // factory = reactor.core.scheduler.ReactorThreadFactory
		this.factory = factory;
           // *******************************
           // 创建双端队列
           // *******************************
		this.cache = new ConcurrentLinkedDeque<>();
		this.all = new ConcurrentLinkedQueue<>();
           // ***************************************
           // 创建回收线程池 core:1 max:Integer.MAX_VALUE
           // ***************************************
		this.evictor = Executors.newScheduledThreadPool(1, EVICTOR_FACTORY);
		// 启动线程回收
           this.evictor.scheduleAtFixedRate(
                           this::eviction, 
                            // 延迟60秒
				ttlSeconds,
                            // 间隔60秒
				ttlSeconds,
                            // 单位为秒
				TimeUnit.SECONDS);
	}
    
}
```
### (6).总结
1. 创建Schedulers.ElasticScheduler,
它的数据结构是:
ThreadFactory(reactor.core.scheduler.ReactorThreadFactory)
ScheduledExecutorService
2. 实际就是创建了<font color='red'>ScheduledExecutorService</font>的子类