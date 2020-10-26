---
layout: post
title: 'Netty源码学习(EventLoop如何处理READ事件)'
date: 2020-10-26
author: 李新
tags: Netty
---

### (1).Netty又是如何处理读写请求的呢?
> 代码的入口在:ServerBootstrap$ServerBootstrapAcceptor.channelRead处.

### (2).ServerBootstrap$ServerBootstrapAcceptor.channelRead
```
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    final Channel child = (Channel) msg;
    
    // ****************************************************
    // 添加childHandler
    // ****************************************************
    child.pipeline().addLast(childHandler);
    
    setChannelOptions(child, childOptions, logger);
    setAttributes(child, childAttrs);
    
    try {
        // ****************************************************
        // ****************************************************
        // ----------------------重点就在于此处------------------
        // 由于BossGroup与WorkGroup都属于:NioEventLoop,所以debug有一些难点.
        // 在内存中是有存在两个NioEventLoop对象的.
        // childGroup(NioEventLoop)注册NioSocketChannel
        // ****************************************************
        // ****************************************************
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
} // end 
```
### (3).如何debug?
```
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
EventLoopGroup workerGroup = new NioEventLoopGroup(1);
```
> EventLoop在内存中有两个对象的,所以,断点都是打在NioEventLoop.run方法里.那到底要如何跟踪呢?  
  可以针对连接和读写分别jstack的跟踪,查看线程的调用链到底是属于BossGroup还是WorkGroup.  
  在NioEventLoop.processSelectedKeysOptimized方法里,增加条件断点.  
    如果k.attachment是:NioServerSocketChannel则事件处理是属于:ACCEPT.  
    如果k.attachment是:NioSocketChannel则事件处理属于:READ/WRITE  
    设置条件条件表达式为:  
      [a.getClass().equals(io.netty.channel.socket.nio.NioSocketChannel.class)]

!["NioServerSocketChannel"](/assets/netty/imgs/debug-1.png)
!["NioSocketChannel"](/assets/netty/imgs/debug-2.png)
!["Eclipse设置条件断点"](/assets/netty/imgs/debug-3.png)

### (4).NioEventLoop.run
```
protected void run() {
    int selectCnt = 0;
    for (;;) {
        // ... ... 
        try {
            // 处理Keys
            processSelectedKeys();
        } finally {
            final long ioTime = System.nanoTime() - ioStartTime;
            ranTasks = runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
       }
       // ... ... 
    }// end for
} // end run
```
### (5).NioEventLoop.processSelectedKeys
```
private void processSelectedKeys() {
    if (selectedKeys != null) { // true
        processSelectedKeysOptimized();
    } else {
        processSelectedKeysPlain(selector.selectedKeys());
    }
}
```
### (6).NioEventLoop.processSelectedKeysOptimized
```
private void processSelectedKeysOptimized() {
    for (int i = 0; i < selectedKeys.size; ++i) {
        final SelectionKey k = selectedKeys.keys[i];
        selectedKeys.keys[i] = null;
        
        // ********************************************************
        // *******************************提示*********************
        // NioServerSocketChannel 代表NioEventLoop处理的是ACCEPT事件
        // NioSocketChannel 代表NioEventLoop处理的是READ/WRITE事件
        // a = NioSocketChannel 
        // ********************************************************
        final Object a = k.attachment();
    
        if (a instanceof AbstractNioChannel) {
            // 处理单个key事件
            processSelectedKey(k, (AbstractNioChannel) a);
        } else {
            @SuppressWarnings("unchecked")
            NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
            processSelectedKey(k, task);
        }
    
        if (needsToSelectAgain) {
            // null out entries in the array to allow to have it GC'ed once the Channel close
            // See https://github.com/netty/netty/issues/2363
            selectedKeys.reset(i + 1);
    
            selectAgain();
            i = -1;
        }
    }
} // end processSelectedKeysOptimized
```

### (7).NioEventLoop.processSelectedKey
```
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    // **************************************************************************
    // unsafe = AbstractNioMessageChannel$NioMessageUnsafe 代表NioEventLoop处理的是ACCEPT事件
    // unsafe = io.netty.channel.socket.nio.NioSocketChannel$NioSocketChannelUnsafe 代表NioEventLoop处理的是READ/WRITE事件
    // 此处unsafe = io.netty.channel.socket.nio.NioSocketChannel$NioSocketChannelUnsafe
    // 此:处处理的是Client发送过来的消息("hello world")
    // **************************************************************************
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
    }
    
    try {
        int readyOps = k.readyOps();
        // We first need to call finishConnect() before try to trigger a read(...) or write(...) as otherwise
        // the NIO JDK channel implementation may throw a NotYetConnectedException.
        if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
            // remove OP_CONNECT as otherwise Selector.select(..) will always return without blocking
            // See https://github.com/netty/netty/issues/924
            int ops = k.interestOps();
            ops &= ~SelectionKey.OP_CONNECT;
            k.interestOps(ops);
    
            unsafe.finishConnect();
        }
    
        // Process OP_WRITE first as we may be able to write some queued buffers and so free memory.
        if ((readyOps & SelectionKey.OP_WRITE) != 0) {
            // Call forceFlush which will also take care of clear the OP_WRITE once there is nothing left to write
            ch.unsafe().forceFlush();
        }
    
        // Also check for readOps of 0 to workaround possible JDK bug which may otherwise lead
        // to a spin loop
        // true
        if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
            // unsafe  = io.netty.channel.socket.nio.NioSocketChannel$NioSocketChannelUnsafe
            // 调用读取消息
            unsafe.read();
        }
    } catch (CancelledKeyException ignored) {
        unsafe.close(unsafe.voidPromise());
    }
}// end 
```
### (8).NioSocketChannel$NioSocketChannelUnsafe.read
```
public final void read() {
    final ChannelConfig config = config();
    if (shouldBreakReadReady(config)) { // false
        clearReadPending();
        return;
    }  // end if
    
    // DefaultChannelPipeline
    // {
    //   (DelimiterBasedFrameDecoder#0 = io.netty.handler.codec.DelimiterBasedFrameDecoder), 
    //   (StringDecoder#0 = io.netty.handler.codec.string.StringDecoder)
    //   (StringEncoder#0 = io.netty.handler.codec.string.StringEncoder)
    //   (TelnetServerHandler#0 = io.netty.example.telnet.TelnetServerHandler)
    // }
    final ChannelPipeline pipeline = pipeline();
    
    // 获取缓存分配管理器
    // allocator = io.netty.buffer.PooledByteBufAllocator
    final ByteBufAllocator allocator = config.getAllocator();
    // allocHandle = io.netty.channel.AdaptiveRecvByteBufAllocator$HandleImpl
    final RecvByteBufAllocator.Handle allocHandle = recvBufAllocHandle();
    allocHandle.reset(config);
    
    ByteBuf byteBuf = null;
    boolean close = false;
    try {
        do {
            // ******************************************************************
            // 创建ByteBuf对象,并从SocketChannel中读取数据到:ByteBuf中
            // ******************************************************************
            // allocHandle = io.netty.channel.AdaptiveRecvByteBufAllocator$HandleImpl
            // 创建一个ByteBuf对象(PooledByteBufAllocator)
            byteBuf = allocHandle.allocate(allocator);
            
            // 读取(SocketChannel)数据到ByteBuf中
            allocHandle.lastBytesRead(doReadBytes(byteBuf));
            if (allocHandle.lastBytesRead() <= 0) {
                // nothing was read. release the buffer.
                byteBuf.release();
                byteBuf = null;
                close = allocHandle.lastBytesRead() < 0;
                if (close) {
                    // There is nothing left to read as we received an EOF.
                    readPending = false;
                }
                break;
            }
    
            allocHandle.incMessagesRead(1);
            readPending = false;
            pipeline.fireChannelRead(byteBuf);
            byteBuf = null;
        } while (allocHandle.continueReading());
    
        allocHandle.readComplete();
        pipeline.fireChannelReadComplete();
    
        if (close) {
            closeOnRead(pipeline);
        }
    } catch (Throwable t) {
        handleReadException(pipeline, byteBuf, t, close, allocHandle);
    } finally {
        // Check if there is a readPending which was not processed yet.
        // This could be for two reasons:
        // * The user called Channel.read() or ChannelHandlerContext.read() in channelRead(...) method
        // * The user called Channel.read() or ChannelHandlerContext.read() in channelReadComplete(...) method
        //
        // See https://github.com/netty/netty/issues/2254
        if (!readPending && !config.isAutoRead()) {
            removeReadOp();
        }
    }
} // end read
```
### (9).总结


