---
layout: post
title: 'Netty源码(IdleStateHandler)'
date: 2020-10-26
author: 李新
tags: Netty
---

### (1).IdleStateHandler
```
package io.netty.handler.timeout;
public class IdleStateHandler 
       extends ChannelDuplexHandler {  //1. 继承于:ChannelDuplexHandler 
}
```
### (2).ChannelDuplexHandler
>   ChannelDuplexHandler属于:ChannelInboundHandlerAdaptert和ChannelOutboundHandler,所以.即可对入站消息进行管理,也可对出站消息进行管理 

```
package io.netty.channel;
public class ChannelDuplexHandler 
       extends ChannelInboundHandlerAdapter 
       implements ChannelOutboundHandler {                        
}
```
### (3).IdleStateHandler 构造器
```
// *************** IdleStateHandler ***************
public IdleStateHandler(
        long readerIdleTime, long writerIdleTime, long allIdleTime,
        TimeUnit unit) {
    this(false, readerIdleTime, writerIdleTime, allIdleTime, unit);
}

```
### (4). IdleStateHandler 构造器
```
public IdleStateHandler(
          boolean observeOutput,
          long readerIdleTime, 
          long writerIdleTime, 
          long allIdleTime,
          TimeUnit unit) {
    // 对参数进行检查
    if (unit == null) {
        throw new NullPointerException("unit");
    }
    this.observeOutput = observeOutput;
    if (readerIdleTime <= 0) {
        readerIdleTimeNanos = 0;
    } else {
        readerIdleTimeNanos = Math.max(unit.toNanos(readerIdleTime), MIN_TIMEOUT_NANOS);
    }
    if (writerIdleTime <= 0) {
        writerIdleTimeNanos = 0;
    } else {
        writerIdleTimeNanos = Math.max(unit.toNanos(writerIdleTime), MIN_TIMEOUT_NANOS);
    }
    if (allIdleTime <= 0) {
        allIdleTimeNanos = 0;
    } else {
        allIdleTimeNanos = Math.max(unit.toNanos(allIdleTime), MIN_TIMEOUT_NANOS);
    }
}
```
### (5).IdleStateHandler.handlerAdded
```
public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
    if (ctx.channel().isActive() && ctx.channel().isRegistered()) {
        // channelActive() event has been fired already, which means this.channelActive() will
        // not be invoked. We have to initialize here instead.
        initialize(ctx);
    } else {
        // channelActive() event has not been fired yet.  this.channelActive() will be invoked
        // and initialization will occur there.
    }
}
```

### (6).IdleStateHandler.initialize 
> initialize()方法会创建相应的定时任务.并提交到EventLoop队列里.

```
private void initialize(ChannelHandlerContext ctx) {
    // Avoid the case where destroy() is called before scheduling timeouts.
    // See: https://github.com/netty/netty/issues/143
    switch (state) {
    case 1:
    case 2:
        return;
    }

    state = 1;
    initOutputChanged(ctx);

    lastReadTime = lastWriteTime = ticksInNanos();
    if (readerIdleTimeNanos > 0) {
        // 创建定时任务
        readerIdleTimeout = schedule(ctx, new ReaderIdleTimeoutTask(ctx),
                readerIdleTimeNanos, TimeUnit.NANOSECONDS);
    }
    if (writerIdleTimeNanos > 0) {
        writerIdleTimeout = schedule(ctx, new WriterIdleTimeoutTask(ctx),
                writerIdleTimeNanos, TimeUnit.NANOSECONDS);
    }
    if (allIdleTimeNanos > 0) {
        allIdleTimeout = schedule(ctx, new AllIdleTimeoutTask(ctx),
                allIdleTimeNanos, TimeUnit.NANOSECONDS);
    }
} //end initialize

ScheduledFuture<?> schedule(ChannelHandlerContext ctx, Runnable task, long delay, TimeUnit unit) {
        return ctx.executor().schedule(task, delay, unit);
} //end schedule
```
### (7). IdleStateHandler.channelRead/channelReadComplete方法
> 在channelRead()/channelReadComplete()方法中会记录最后一次读取的时间

```
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {    
    // 如果:读空闲时间>0或者所有的空闲时间>0
    if (readerIdleTimeNanos > 0 || allIdleTimeNanos > 0) {
            // 标记(reading ) = true
            // firstReaderIdleEvent = true
            // firstAllIdleEvent = true
            reading = true;
            firstReaderIdleEvent = firstAllIdleEvent = true;
    }
    // 透传给下一个ChannelHandler
    ctx.fireChannelRead(msg);
}


public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
    // (如果读空闲时间>0或者所有的时间>0 ) 并且 标记(reading)为True的情况下
    if ((readerIdleTimeNanos > 0 || allIdleTimeNanos > 0) && reading) {            
            // *************************************
            // 每调用一次:channelReadComplete,则记录这次调用的时间
            lastReadTime = ticksInNanos();
            reading = false;
    }
    ctx.fireChannelReadComplete();
}
```

### (8).IdleStateHandler$ReaderIdleTimeoutTask 

```
private abstract static class AbstractIdleTask implements Runnable {
    private final ChannelHandlerContext ctx;

    AbstractIdleTask(ChannelHandlerContext ctx) {
        this.ctx = ctx;
    }
    
    @Override
    public void run() {
        // 如果channel没有被打开
        if (!ctx.channel().isOpen()) {
            return;
        }
        run(ctx);
    }
    protected abstract void run(ChannelHandlerContext ctx);
} //end AbstractIdleTask 

private final class ReaderIdleTimeoutTask extends AbstractIdleTask {
    ReaderIdleTimeoutTask(ChannelHandlerContext ctx) {
        super(ctx);
    }
    @Override
    protected void run(ChannelHandlerContext ctx) {
        // nextDelay = 下一次进行调度的时间(假设为:1分钟)
        long nextDelay = readerIdleTimeNanos;
        if (!reading) { //有调用过:channelReadComplete方法,reading则为:false
            // *************************************
            // 下一次延迟调度时间 = 调度时间(1分钟) - (当前进间 - 上一次调度的时间)                        
            nextDelay -= ticksInNanos() - lastReadTime;
        }

        if (nextDelay <= 0) {
            // 设置下一次调度
            // Reader is idle - set a new timeout and notify the callback.
            readerIdleTimeout = schedule(ctx, this, readerIdleTimeNanos, TimeUnit.NANOSECONDS);
            // 判断是否为第一次产生事件
            boolean first = firstReaderIdleEvent;
            firstReaderIdleEvent = false;

            try {
                // 创建事件
                IdleStateEvent event = newIdleStateEvent(IdleState.READER_IDLE, first);
                // 触发空闲事件
                channelIdle(ctx, event);
            } catch (Throwable t) {
                ctx.fireExceptionCaught(t);
            }
        } else {
            // Read occurred before the timeout - set a new timeout with shorter delay.
            readerIdleTimeout = schedule(ctx, this, nextDelay, TimeUnit.NANOSECONDS);
        }
    }
}
```
### (9).IdleStateHandler newIdleStateEvent创建事件
```
protected IdleStateEvent newIdleStateEvent(IdleState state, boolean first) {
    switch (state) {
        case ALL_IDLE:
            return first ? IdleStateEvent.FIRST_ALL_IDLE_STATE_EVENT : IdleStateEvent.ALL_IDLE_STATE_EVENT;
        case READER_IDLE:
            return first ? IdleStateEvent.FIRST_READER_IDLE_STATE_EVENT : IdleStateEvent.READER_IDLE_STATE_EVENT;
        case WRITER_IDLE:
            return first ? IdleStateEvent.FIRST_WRITER_IDLE_STATE_EVENT : IdleStateEvent.WRITER_IDLE_STATE_EVENT;
        default:
            throw new IllegalArgumentException("Unhandled: state=" + state + ", first=" + first);
    }
}
```
### (10).触发空闲事件
```
protected void channelIdle(ChannelHandlerContext ctx, IdleStateEvent evt) throws Exception {
    ctx.fireUserEventTriggered(evt);
}
```
### (11).总结
> 1. 为每一个Channel添加IdleStateHandler,并添加定时任务
> 2. 当Channel发生读/写时,记录最后的读/写时间
> 3. 定时任务检查:最后(读/写)的时间是否大于(IdleStateHandler构造器)配置的时间
> 4. 如果超出配置的时间,则产生相应的事件(READER_IDLE/WRITER_IDLE/ALL_IDLE).
> 5. 继续计算下一次定时任务的时间