---
layout: post
title: 'Netty源码学习(EventLoopGroup)'
date: 2020-10-26
author: 李新
tags: Netty
---

### (1).new NioEventLoopGroup
```
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
```
### (2).NioEventLoopGroup 构造器
```
public NioEventLoopGroup(int nThreads) {
    // nThreads = 1
    this(nThreads, (Executor) null);
}
```

### (3).NioEventLoopGroup 构造器
```
public NioEventLoopGroup(int nThreads, Executor executor) {
    // nThreads = 1
    // executor = null
    // 调用:SelectorProvider.provider() 获得一个:SelectorProvider
    this(nThreads, executor, SelectorProvider.provider());
}
```
### (4).NioEventLoopGroup 构造器
```
public NioEventLoopGroup(int nThreads, Executor executor, final SelectorProvider selectorProvider) {
    // nThreads = 1
    // executor = null
    // 因为是mac:
    // selectorProvider = sun.nio.ch.KQueueSelectorProvider
    this(nThreads, executor, selectorProvider, DefaultSelectStrategyFactory.INSTANCE);
}
```
### (5).NioEventLoopGroup 构造器
```
public NioEventLoopGroup(int nThreads, Executor executor, final SelectorProvider selectorProvider,final SelectStrategyFactory selectStrategyFactory) {
    // nThreads = 1
    // executor = null
    // selectorProvider = sun.nio.ch.KQueueSelectorProvider
    // selectStrategyFactory = io.netty.channel.DefaultSelectStrategyFactory
    // 设置拒绝策略
    // 调用:MultithreadEventLoopGroup的构造器
    super(nThreads, executor, selectorProvider, selectStrategyFactory, RejectedExecutionHandlers.reject());
}
```

### (6).MultithreadEventLoopGroup 构造器
```
protected MultithreadEventLoopGroup(int nThreads, Executor executor, Object... args) {
    // nThreads = 1
    // executor = null
    // args = SelectorProvider/SelectStrategyFactory/RejectedExecutionHandlers$1
    // args = [sun.nio.ch.KQueueSelectorProvider, io.netty.channel.DefaultSelectStrategyFactory, io.netty.util.concurrent.RejectedExecutionHandlers$1]
    // 调用父类MultithreadEventExecutorGroup的构造器
    super(nThreads == 0 ? DEFAULT_EVENT_LOOP_THREADS : nThreads, executor, args);
}
```
### (7).MultithreadEventExecutorGroup 构造器
```
protected MultithreadEventExecutorGroup(int nThreads, Executor executor, Object... args) {
    // nThreads = 1
    // executor = null
    // args = [sun.nio.ch.KQueueSelectorProvider, io.netty.channel.DefaultSelectStrategyFactory, io.netty.util.concurrent.RejectedExecutionHandlers$1]
    this(nThreads, executor, DefaultEventExecutorChooserFactory.INSTANCE, args);
}
```
### (8).MultithreadEventExecutorGroup 构造器
```
// 属性域
EventExecutor[] children;
EventExecutorChooserFactory.EventExecutorChooser chooser;
AtomicInteger terminatedChildren = new AtomicInteger();
Promise<?> terminationFuture = new DefaultPromise(GlobalEventExecutor.INSTANCE);

// ********************************************
// 重点:
// ********************************************
protected MultithreadEventExecutorGroup(
            int nThreads, 
            Executor executor,
            EventExecutorChooserFactory chooserFactory, 
            Object... args) {
    // nThreads = 1
    // executor = null
    // chooserFactory = io.netty.util.concurrent.DefaultEventExecutorChooserFactory
    // args = [
   //   sun.nio.ch.KQueueSelectorProvider
   //   io.netty.channel.DefaultSelectStrategyFactory
   //   io.netty.util.concurrent.RejectedExecutionHandlers$1
   //]
    if (nThreads <= 0) {  // false
        throw new IllegalArgumentException(String.format("nThreads: %d (expected: > 0)", nThreads));
    }
    
    // 用户没有指定线程池的情况下
    if (executor == null) { // true
        // 创建自定义的:Executor
        // 另开一节讲这部份的源码,先不纠结里面是什么
        executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
    }

    // EventExecutor[] children 为定义部份
    // 创建EventExecutor数组
    children = new EventExecutor[nThreads];

    for (int i = 0; i < nThreads; i ++) {
        boolean success = false;
        try {
            // ************************************************************
            // 先忽略,后面详解
            // 1. 委托给子类:NioEventLoopGroup去执行
            // [io.netty.channel.nio.NioEventLoop@6b81ce95]
            // ************************************************************
            children[i] = newChild(executor, args);
            success = true;
        } catch (Exception e) {
            // TODO: Think about if this is a good exception type
            throw new IllegalStateException("failed to create a child event loop", e);
        } finally {
            if (!success) { // 执行不成功的处理
                for (int j = 0; j < i; j ++) {
                        children[j].shutdownGracefully();
                } //end for

                for (int j = 0; j < i; j ++) {
                    EventExecutor e = children[j];
                    try {
                        while (!e.isTerminated()) {
                            e.awaitTermination(Integer.MAX_VALUE, TimeUnit.SECONDS);
                        } //end while
                    } catch (InterruptedException interrupted) {
                        Thread.currentThread().interrupt();
                        break;
                    } //end catch
                }// end for
            } // end if
        } // finally
    } //end for

    // ************************************************************
    // 先忽略,后面详解
    //  chooser是一个选择器,可以从childern(NioEventLoop[])中选择一个:EventExecutor
    // chooserFactory.io.netty.util.concurrent.DefaultEventExecutorChooserFactory
    // chooser = io.netty.util.concurrent.DefaultEventExecutorChooserFactory$PowerOfTwoEventExecutorChooser
    // ************************************************************
    chooser = chooserFactory.newChooser(children);

    // 创建中止监听器
    final FutureListener<Object> terminationListener = new FutureListener<Object>() {
        @Override
        public void operationComplete(Future<Object> future) throws Exception {
            if (terminatedChildren.incrementAndGet() == children.length) {
                terminationFuture.setSuccess(null);
            }
        }
    }; //end 

    // 为所有的:NioEventLoop添加中止监听器
    for (EventExecutor e: children) {
            e.terminationFuture().addListener(terminationListener);
    }

    // 把NioEventLoop数组转换成Set集合.
    Set<EventExecutor> childrenSet = new LinkedHashSet<EventExecutor>(children.length);
    Collections.addAll(childrenSet, children);
    // 设置为只读的NioEventLoop
    readonlyChildren = Collections.unmodifiableSet(childrenSet);
}
```
### (9).NioEventLoopGroup.newChild
```
protected EventLoop newChild(Executor executor, Object... args) throws Exception {
    // executor = io.netty.util.concurrent.ThreadPerTaskExecutor
    // args = [
    //   sun.nio.ch.KQueueSelectorProvider
    //   io.netty.channel.DefaultSelectStrategyFactory
    //   io.netty.util.concurrent.RejectedExecutionHandlers$1
    // ]
    //  queueFactory = null;
    EventLoopTaskQueueFactory queueFactory = args.length == 4 ? (EventLoopTaskQueueFactory) args[3] : null;
    // ********************************************
    // 后面留一节专门讲NioEventLoop,先略过
    // ********************************************
    return new NioEventLoop(this, executor, (SelectorProvider) args[0],((SelectStrategyFactory) args[1]).newSelectStrategy(), (RejectedExecutionHandler) args[2], queueFactory);
}
```
### (10).总结
> NioEventLoopGroup实际是根据配置的nThreads,**创建N个NioEventLoop**
NioEventLoopGroup和NioEventLoop都实现了:EventLoopGroup,相当于间两个类都间接的实现了ScheduledExecutorService(定时调度方法)和Iterable(迭代器).  
**而NioEventLoopGroup的大部份调度方法都是委派给:NioEventLoop去调用的.自己实际不执行任务定时任务的内容.**