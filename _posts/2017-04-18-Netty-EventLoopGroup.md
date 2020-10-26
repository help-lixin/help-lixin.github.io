---
layout: post
title: 'Netty源码学习之-EventLoopGroup'
date: 2020-10-26
author: 李新
tags: Netty
---

### (1).查看NioEventLoopGroup实现关系
```
// 注意EventExecutorGroup
// 实现:java底层的:ScheduledExecutorService和Iterable
// 意味着有遍历和定时调度功能
io.netty.util.concurrent.EventExecutorGroup 
        implements 
        java.util.concurrent.ScheduledExecutorService,
        java.lang.Iterable        

io.netty.channel.EventLoopGroup 
        implements 
        io.netty.util.concurrent.EventExecutorGroup
        
io.netty.channel.MultithreadEventLoopGroup 
        extends 
        io.netty.util.concurrent.MultithreadEventExecutorGroup
        implements 
        io.netty.channel.EventLoopGroup        

io.netty.channel.nio.NioEventLoopGroup 
        extends 
        MultithreadEventLoopGroup

```
> 从NioEventLoopGroup的继承(实现)关系,可以看到:NioEventLoopGroup实现了JDK的:ScheduledExecutorService和Iterable.意味着:NioEventLoopGroup内部有定时任务功能和遍历功能.

### (2).查看接口EventExecutorGroup,包含有哪些行为?
```
public interface EventExecutorGroup 
       // JDK自带的定时调度
       extends ScheduledExecutorService, 
       // JDK的容器遍历
       Iterable<EventExecutor> {
    
    boolean isShuttingDown();
    Future<?> shutdownGracefully();
    Future<?> shutdownGracefully(long quietPeriod, long timeout, TimeUnit unit);
   Future<?> terminationFuture();
   
   // ***************************************************************
   // Iterable行为方法
   EventExecutor next();
   // Iterable行为方法
   Iterator<EventExecutor> iterator();
   // ***************************************************************
   
   Future<?> submit(Runnable task);
   <T> Future<T> submit(Runnable task, T result);
   <T> Future<T> submit(Callable<T> task);
   ScheduledFuture<?> schedule(Runnable command, long delay, TimeUnit unit);
   <V> ScheduledFuture<V> schedule(Callable<V> callable, long delay, TimeUnit unit);
   ScheduledFuture<?> scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnit unit);
   ScheduledFuture<?> scheduleWithFixedDelay(Runnable command, long initialDelay, long delay, TimeUnit unit);
}
```
> EventExecutorGroup定义了遍历EventExecutor的方法,以及提交定时任务的方法.

### (3).查看EventLoopGroup接口,具有哪些行为?
```
public interface EventLoopGroup extends EventExecutorGroup {
    EventLoop next();
    // 
    ChannelFuture register(Channel channel);
    ChannelFuture register(ChannelPromise promise);
    ChannelFuture register(Channel channel, ChannelPromise promise);
}
```
> EventLoopGroup继承于:EventExecutorGroup(定时任务调度/遍历行为),并且定义了注册Channel的抽象行为.

### (4).分析MultithreadEventLoopGroup行为
> MultithreadEventLoopGroup继承于MultithreadEventExecutorGroup,并实现了:EventLoopGroup.
这意味着:MultithreadEventLoopGroup需要实现:定时任务调度/注册Channel/遍历事件等行为.

### (5).总结
> 从NioEventLoopGroup的接口行为来看.NioEventLoopGroup具备有:定时任务度功能/注册Channerl/遍历事件功能等