---
layout: post
title: 'Netty源码(ChannelPipeline)'
date: 2020-10-26
author: 李新
tags: Netty
---

### (1).ChannelPipeline
    说ChannelPipeline时,不得不说下以下几个接口:
        io.netty.channel.ChannelPipeline 
        io.netty.channel.ChannelHandlerContext 
        io.netty.channel.ChannelHandler 
        io.netty.channel.Channel

### (2).ChannelPipeline 接口声明
!["ChannelPipeline接口声明"](/assets/netty/imgs/ChannelPipeline.png)

### (3).ChannelHandlerContext接口详解
!["ChannelHandlerContext接口详解"](/assets/netty/imgs/ChannelHandlerContext.png)

### (4).ChannelHandler接口详解
!["ChannelHandler接口详解"](/assets/netty/imgs/ChannelHandler.png)

### (5).ChannelPipeline.addLast(xxx) 流程图解
!["ChannelPipeline.addLast(xxx) 流程图解"](/assets/netty/imgs/ChannelPipeline-addLast.png)

### (6).ChannelPipeline.addLast(xxx)
```
public class DefaultChannelPipeline implements ChannelPipeline {
    private boolean registered;
    
    // 添加ChanndlHandler
    public final ChannelPipeline addLast(ChannelHandler... handlers) {
        return addLast(null, handlers);
    } // end addLast
    
    public final ChannelPipeline addLast(EventExecutorGroup executor, ChannelHandler... handlers) {
        ObjectUtil.checkNotNull(handlers, "handlers");

        for (ChannelHandler h: handlers) {
            if (h == null) {
                break;
            }
            // addLast
            addLast(executor, null, h);
        }
        return this;
    } // end addLast
    
    public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
        final AbstractChannelHandlerContext newCtx;
        synchronized (this) {
            // 验证
            checkMultiplicity(handler);

            // 创建:DefaultChannelHandlerContext(包裹着channel/handler)
            newCtx = newContext(group, filterName(name, handler), handler);

            // 向链表的尾部加入上下文
            addLast0(newCtx);

            // !(false)
            if (!registered) { 
                // 为新创建的上下文设置状态为:ADD_PEDING
                newCtx.setAddPending();
                // 
                callHandlerCallbackLater(newCtx, true);
                // 直接return了
                return this;
            }

            EventExecutor executor = newCtx.executor();
            if (!executor.inEventLoop()) {
                callHandlerAddedInEventLoop(newCtx, executor);
                return this;
            }
        }
        callHandlerAdded0(newCtx);
        return this;
    } // end addLast
    
    
    private void callHandlerCallbackLater(AbstractChannelHandlerContext ctx, boolean added) {
        assert !registered;
        // added = true
        // 创建:PendingHandlerAddedTask
        PendingHandlerCallback task = added ? new PendingHandlerAddedTask(ctx) : new PendingHandlerRemovedTask(ctx);
        PendingHandlerCallback pending = pendingHandlerCallbackHead;
        if (pending == null) {
            // 设置当前对象的属性:pendingHandlerCallbackHead为:PendingHandlerAddedTask
            pendingHandlerCallbackHead = task;
        } else {
            while (pending.next != null) {
                pending = pending.next;
            }
            pending.next = task;
        }
    }// end callHandlerCallbackLater
    
}
```

### (7).总结
ChannelPipeline.addLast()方法在执行时.
1. 创建DefaultChannelHandlerContext包裹着:ChannelHandler
2. 把DefaultChannelHandlerContext添加到链表的末端.
3. 设置DefaultChannelHandlerContext状态为:ADD_PEDING
4. 创建:PendingHandlerAddedTask任务.