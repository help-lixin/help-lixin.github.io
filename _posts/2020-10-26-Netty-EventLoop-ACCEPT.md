---
layout: post
title: 'Netty源码(EventLoop如何处理ACCEPT事件)'
date: 2020-10-26
author: 李新
tags: Netty
---

### (1).BossGroup(NioEventLoop)初始化完成了,是怎么接受请求(ACCEPT事件),并把请求分发给:WorkGroup(NioEventLoop)的呢?
> 答案在:<font color='red'>NioEventLoop.run()</font>方法里.

### (2).NioEventLoop.run
```
public final class NioEventLoop extends SingleThreadEventLoop {
    protected void run() {
        int selectCnt = 0;
        for (;;) {
            
            try {
                int strategy;
                try {
                    // strategy = -1;
                    strategy = selectStrategy.calculateStrategy(selectNowSupplier, hasTasks());
                    switch (strategy) {
                    case SelectStrategy.CONTINUE:
                        continue;
                    case SelectStrategy.BUSY_WAIT:
                    // ******************************************
                    // 产生轮询
                    // ******************************************
                    case SelectStrategy.SELECT:
                        // curDeadlineNanos = -1
                        long curDeadlineNanos = nextScheduledTaskDeadlineNanos();
                        if (curDeadlineNanos == -1L) { // true
                            //  NONE = 9223372036854775807
                            curDeadlineNanos = NONE; // nothing on the calendar
                        }
                        // 
                        nextWakeupNanos.set(curDeadlineNanos);
                        try {
                            if (!hasTasks()) { //true
                                // **************************************
                                // strategy = -1
                                // 3. 调用Selector.select() 阻塞等待客户端的连接
                                // **************************************
                                strategy = select(curDeadlineNanos);
                            }
                        } finally {
                            nextWakeupNanos.lazySet(AWAKE);
                        }
                        // fall through
                    default:
                    }
                } catch (IOException e) {
                    rebuildSelector0();
                    selectCnt = 0;
                    handleLoopException(e);
                    continue;
                } //end catch

                selectCnt++;
                cancelledKeys = 0;
                needsToSelectAgain = false;
                final int ioRatio = this.ioRatio;
                boolean ranTasks;
                
                // ioRatio = 50
                if (ioRatio == 100) {
                    try {
                        if (strategy > 0) {
                            processSelectedKeys();
                        }
                    } finally {
                        // Ensure we always run tasks.
                        ranTasks = runAllTasks();
                    }
                } else if (strategy > 0) {
                    final long ioStartTime = System.nanoTime();
                    try {
                        // ******************************************
                        // 4. 处理Select事件
                        // ******************************************
                        processSelectedKeys();
                    } finally {
                        // Ensure we always run tasks.
                        // ioTime = 650326301514
                        final long ioTime = System.nanoTime() - ioStartTime;
                        // **************************************************
                        // 9. 运行所有的任务(SingleThreadEventExecutor.runAllTasks)
                        // **************************************************
                        ranTasks = runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                    }
                } else {
                    ranTasks = runAllTasks(0); // This will run the minimum number of tasks
                }

                if (ranTasks || strategy > 0) {
                    if (selectCnt > MIN_PREMATURE_SELECTOR_RETURNS && logger.isDebugEnabled()) {
                        logger.debug("Selector.select() returned prematurely {} times in a row for Selector {}.",
                                selectCnt - 1, selector);
                    }
                    selectCnt = 0;
                } else if (unexpectedSelectorWakeup(selectCnt)) { // Unexpected wakeup (unusual case)
                    selectCnt = 0;
                }
            } catch (CancelledKeyException e) {
                // Harmless exception - log anyway
                if (logger.isDebugEnabled()) {
                    logger.debug(CancelledKeyException.class.getSimpleName() + " raised by a Selector {} - JDK bug?",
                            selector, e);
                }
            } catch (Throwable t) {
                handleLoopException(t);
            }
            
            // Always handle shutdown even if the loop processing threw an exception.
            try {
                if (isShuttingDown()) {
                    closeAll();
                    if (confirmShutdown()) {
                        return;
                    }
                }
            } catch (Throwable t) {
                handleLoopException(t);
            }// end catch
            
        } // end for
    } // end run
}
```
### (3).NioEventLoop.select
```
private int select(long deadlineNanos) throws IOException {
    // deadlineNanos = 9223372036854775807
    if (deadlineNanos == NONE) { // true
        // selector  = io.netty.channel.nio.SelectedSelectionKeySetSelector
        // ************************************************
        //  select()为阻塞方法
        //  阻塞等待客户端的连接.一旦发现有连接过来,就将事件存储到:
        //  SelectedSelectionKeySet里面
        // ************************************************
        return selector.select();
    }
    // Timeout will only be 0 if deadline is within 5 microsecs
    long timeoutMillis = deadlineToDelayNanos(deadlineNanos + 995000L) / 1000000L;
    return timeoutMillis <= 0 ? selector.selectNow() : selector.select(timeoutMillis);
}
```
### (4).NioEventLoop.processSelectedKeys
```
private void processSelectedKeys() {
    // selectedKeys = sun.nio.ch.SelectionKeyImpl
    if (selectedKeys != null) {
        processSelectedKeysOptimized();
    } else {
        processSelectedKeysPlain(selector.selectedKeys());
    }
}
```
### (5). NioEventLoop.processSelectedKeysOptimized
```
private void processSelectedKeysOptimized() {
    // *************************************************
    // NioEventLoop.select() 方法会将用户触发的事件添加到:
    // selectedKeys中
    // *************************************************
    // selectedKeys.size = 1 此时只有一个client连接上来了
    for (int i = 0; i < selectedKeys.size; ++i) {
        // k = sun.nio.ch.SelectionKeyImpl
        // 获得对应下标的SelectionKey
        final SelectionKey k = selectedKeys.keys[i];
        // 立即将对应的key设置为空
        selectedKeys.keys[i] = null;
        
        // a = NioServerSocketChannel
        final Object a = k.attachment();
        
        if (a instanceof AbstractNioChannel) { // true
            processSelectedKey(k, (AbstractNioChannel) a);
        } else {
            @SuppressWarnings("unchecked")
            NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
            processSelectedKey(k, task);
        }
    
        if (needsToSelectAgain) { // false
            // null out entries in the array to allow to have it GC'ed once the Channel close
            // See https://github.com/netty/netty/issues/2363
            selectedKeys.reset(i + 1);
    
            selectAgain();
            i = -1;
        }
    }
}
```
### (6).NioEventLoop.processSelectedKey
```
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    //  k = sun.nio.ch.SelectionKeyImpl
    // ch = NioServerSocketChannel 
    
    // 获得NioServerSocketChannel内部的Unsafe对象
    // unsafe = AbstractNioMessageChannel$NioMessageUnsafe
    final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();

    if (!k.isValid()) { // false
        final EventLoop eventLoop;
        try {
            eventLoop = ch.eventLoop();
        } catch (Throwable ignored) {
            // If the channel implementation throws an exception because there is no event loop, we ignore this
            // because we are only trying to determine if ch is registered to this event loop and thus has authority
            // to close ch.
            return;
        }
        // Only close ch if ch is still registered to this EventLoop. ch could have deregistered from the event loop
        // and thus the SelectionKey could be cancelled as part of the deregistration process, but the channel is
        // still healthy and should not be closed.
        // See https://github.com/netty/netty/issues/5125
        if (eventLoop == this) {
            // close the channel if the key is not valid anymore
            unsafe.close(unsafe.voidPromise());
        }
        return;
    }// end if
    
    try {
        // readyOps = 16
        int readyOps = k.readyOps();
        // We first need to call finishConnect() before try to trigger a read(...) or write(...) as otherwise
        // the NIO JDK channel implementation may throw a NotYetConnectedException.
        if ((readyOps & SelectionKey.OP_CONNECT) != 0) { // false
            // remove OP_CONNECT as otherwise Selector.select(..) will always return without blocking
            // See https://github.com/netty/netty/issues/924
            int ops = k.interestOps();
            ops &= ~SelectionKey.OP_CONNECT;
            k.interestOps(ops);
    
            unsafe.finishConnect();
        }
    
        // Process OP_WRITE first as we may be able to write some queued buffers and so free memory.
        if ((readyOps & SelectionKey.OP_WRITE) != 0) { // false
            // Call forceFlush which will also take care of clear the OP_WRITE once there is nothing left to write
            ch.unsafe().forceFlush();
        }
    
        // Also check for readOps of 0 to workaround possible JDK bug which may otherwise lead
        // to a spin loop
        // true
        if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
            // *****************************************************
            // 6. AbstractNioMessageChannel.NioMessageUnsafe.read()
            // *****************************************************
            unsafe.read();
        }
    } catch (CancelledKeyException ignored) {
        unsafe.close(unsafe.voidPromise());
    }
}
```
### (7).AbstractNioMessageChannel.NioMessageUnsafe.read
```
public void read() {
    assert eventLoop().inEventLoop();
    // config = NioServerSocketChannel$NioServerSocketChannelConfig
    final ChannelConfig config = config();
    // 一个日志ChannelHandler 和 ServerBootstrap$ServerBootstrapAcceptor
    // pipeline = DefaultChannelPipeline{(LoggingHandler#0 = io.netty.handler.logging.LoggingHandler), (ServerBootstrap$ServerBootstrapAcceptor#0 = io.netty.bootstrap.ServerBootstrap$ServerBootstrapAcceptor)}
    final ChannelPipeline pipeline = pipeline();
    
    // allocHandle = io.netty.channel.AdaptiveRecvByteBufAllocator$HandleImpl
    final RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle();
    allocHandle.reset(config);
    
        
    boolean closed = false;
    Throwable exception = null;
    try {
        try {
            do {
                // 调用:NioServerSocketChannel.doReadMessages
                // 读取消息
                int localRead = doReadMessages(readBuf);
                
                if (localRead == 0) { // false
                    break;
                }
                
                if (localRead < 0) { // false
                    closed = true;
                    break;
                }
                
                
                // allocHandler = io.netty.channel.AdaptiveRecvByteBufAllocator$HandleImpl
                allocHandle.incMessagesRead(localRead);
            } while (allocHandle.continueReading()); // end do while
        } catch (Throwable t) {
            exception = t;
        } // end catch

        
        int size = readBuf.size();
        for (int i = 0; i < size; i ++) {
            readPending = false;
            // ******************************************************
            // readBuf.get(i) = NioSocketChannel
            // 调用BoosGroup里配置的所有handler.fireChannelRead
            // 传递的NioSocketChannel是刚才:doReadMessages new出来的
            // ******************************************************
            // 8.ServerBootstrap$ServerBootstrapAcceptor.channelRead()
            pipeline.fireChannelRead(readBuf.get(i));
        } // end for
        
        // 清除集合中的数据
        readBuf.clear();
        // 标记读取完成
        allocHandle.readComplete();
        
        // ***************************************************
        // 调用pipeline上所有的ChannelHandler.channelReadComplete(ctx) 方法
        // ***************************************************
        pipeline.fireChannelReadComplete();
    
        if (exception != null) { // false
            closed = closeOnReadError(exception);
            pipeline.fireExceptionCaught(exception);
        }// end if
    
        if (closed) { // false
            inputShutdown = true;
            if (isOpen()) {
                close(voidPromise());
            } // end if
        }// end if
    } finally {
        // Check if there is a readPending which was not processed yet.
        // This could be for two reasons:
        // * The user called Channel.read() or ChannelHandlerContext.read() in channelRead(...) method
        // * The user called Channel.read() or ChannelHandlerContext.read() in channelReadComplete(...) method
        //
        // See https://github.com/netty/netty/issues/2254

        if (!readPending && !config.isAutoRead()) { // false
            removeReadOp();
        }
    } // end finally
}// end read
```
### (8).NioServerSocketChannel.doReadMessages
```
protected int doReadMessages(List<Object> buf) throws Exception {
    // buf=[]
    
    // **************************************************************
    // javaChannel = sun.nio.ch.ServerSocketChannelImpl
    // 调用:ServerSocketChannel.accept()方法
    // 返回一个:java.nio.channels.SocketChannel对象
    // **************************************************************
    java.nio.channels.SocketChannel ch = SocketUtils.accept(javaChannel());
    
    try {
        if (ch != null) {
            // 创建一个:io.netty.channel.socket.nio.NioSocketChannel
            // NioSocketChannel 内部持
            // Channel = NioServerSocketChannel
            // SocketChannel = SocketChannel
            // 添加到buf中
            buf.add(new NioSocketChannel(this, ch));
            return 1;
        }
    } catch (Throwable t) {
        logger.warn("Failed to create a new channel from an accepted socket.", t); 
        try {
            ch.close();
        } catch (Throwable t2) {
            logger.warn("Failed to close a socket.", t2);
        }
    }    
    return 0;
}
```
### (9).ServerBootstrap$ServerBootstrapAcceptor.channelRead
```
private final EventLoopGroup childGroup;
private final ChannelHandler childHandler;
private final Entry<ChannelOption<?>, Object>[] childOptions;
private final Entry<AttributeKey<?>, Object>[] childAttrs;
private final Runnable enableAutoReadTask;

public void channelRead(ChannelHandlerContext ctx, Object msg) {
    // msg = NioSocketChannel
    // ctx = DefaultChannelHandlerContext
    final Channel child = (Channel) msg;
    
    // 为WorkGroup配置管道信息
    child.pipeline().addLast(childHandler);
    // 为WorkGroup配置options
    setChannelOptions(child, childOptions, logger);
    // 为WorkGroup配置atrributes
    setAttributes(child, childAttrs);
    
    try {
        // 向WorkGroup(EventLoopGroup[EventLoop])注册Channel
        childGroup.register(child).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                if (!future.isSuccess()) {
                    forceClose(child, future.cause());
                }
            }
        });
    } catch (Throwable t) {
        forceClose(child, t);
    }
} // end channelRead
```
### (10).NioEventLoop.runAllTasks
```
protected boolean runAllTasks(long timeoutNanos) {
    // timeoutNanos = 650326301514
    
    fetchFromScheduledTaskQueue();
    // 拉取任务
    Runnable task = pollTask();
    if (task == null) { // true
        afterRunningAllTasks();
        return false;
    }
    
    final long deadline = timeoutNanos > 0 ? ScheduledFutureTask.nanoTime() + timeoutNanos : 0;
    long runTasks = 0;
    long lastExecutionTime;
    for (;;) {
        safeExecute(task);
    
        runTasks ++;
    
        // Check timeout every 64 tasks because nanoTime() is relatively expensive.
        // XXX: Hard-coded value - will make it configurable if it is really a problem.
        if ((runTasks & 0x3F) == 0) {
            lastExecutionTime = ScheduledFutureTask.nanoTime();
            if (lastExecutionTime >= deadline) {
                break;
            }
        }
    
        task = pollTask();
        if (task == null) {
            lastExecutionTime = ScheduledFutureTask.nanoTime();
            break;
        }
    }
    
    afterRunningAllTasks();
    this.lastExecutionTime = lastExecutionTime;
    return true;
}
```
### (11).ACCEPT过程图解
!["ACCEPT过程图解"](/assets/netty/imgs/ServerSocketChannel-ACCEPT-EVENT.png)

### (12).总结
1. NioEventLoop.run方法会不停的轮询.当返回的策略为:SelectStrategy.SELECT
2. 调用:SelectedSelectionKeySetSelector.select()方法,会阻塞方法继续往下执行,直到有事件进入.并把事件存储到:SelectedSelectionKeySet集合中(修改Selector的事件存储对象).
3. 遍历集合(SelectedSelectionKeySet)中所有的事件(SelectionKey.OP_ACCEPT)
4. 调用:AbstractNioMessageChannel.NioMessageUnsafe.read处理ACCEPT/READ事件
5. 调用:NioServerSocketChannel.doReadMessages方法,会委托给:ServerSocketChannel.accept,返回一个SocketChannel.
6. 调用:DefaultChannelPipeline.fireChannelRead,通知所有ChannelHandler.channelRead方法(ServerBootstrap.ServerBootstrapAcceptor.channelRead)
    6.1 为WorkGroup(EventLoop)添加ChannelHandler
    6.2 为WorkGroup(EventLoop)注册SocketChannel(子Channel)
7. 调用:DefaultChannelPipeline.fireChannelReadComplete.通知所有ChannelHandler.channelReadComplete方法