---
layout: post
title: 'Netty源码学习(EventLoop)'
date: 2020-10-26
author: 李新
tags: Netty
---

### (1).NioEventLoopGroup.newChild
```
protected EventLoop newChild(Executor executor, Object... args) throws Exception {
    // args = [
    // sun.nio.ch.KQueueSelectorProvider
    // io.netty.channel.DefaultSelectStrategyFactory
    // io.netty.util.concurrent.RejectedExecutionHandlers$1
    // ]
    EventLoopTaskQueueFactory queueFactory = args.length == 4 ? (EventLoopTaskQueueFactory) args[3] : null;
    return new NioEventLoop( 
           // NioEventLoopGroup 
           this, 
           // io.netty.util.concurrent.ThreadPerTaskExecutor
           executor, 
           // sun.nio.ch.KQueueSelectorProvider
           (SelectorProvider) args[0],
           // io.netty.channel.DefaultSelectStrategyFactory
           ((SelectStrategyFactory) args[1]).newSelectStrategy(), 
           // io.netty.util.concurrent.RejectedExecutionHandlers$1
           (RejectedExecutionHandler) args[2], 
           // null
           queueFactory
    ); //end create NioEventLoop
}
```

### (2).NioEventLoop构造器
```
NioEventLoop(
        // parent = io.netty.channel.nio.NioEventLoopGroup
        NioEventLoopGroup parent, 
        // executor = io.netty.util.concurrent.ThreadPerTaskExecutor
        Executor executor, 
        // selectorProvider = sun.nio.ch.KQueueSelectorProvider
        SelectorProvider selectorProvider,
        // strategy = io.netty.channel.DefaultSelectStrategy
        SelectStrategy strategy, 
       // rejectedExecutionHandler = io.netty.util.concurrent.RejectedExecutionHandlers$1
       RejectedExecutionHandler rejectedExecutionHandler,
       // queueFactory = null
       EventLoopTaskQueueFactory queueFactory) {
    
    // 调用父类:SingleThreadEventLoop构造器
    super(    
        parent, 
        executor, 
        false, 
        // ********************************************
       newTaskQueue(queueFactory), 
       newTaskQueue(queueFactory),
       // ********************************************
        rejectedExecutionHandler
    ); // end super
    
    // sun.nio.ch.KQueueSelectorProvider
    this.provider = ObjectUtil.checkNotNull(selectorProvider, "selectorProvider");
    // io.netty.channel.DefaultSelectStrategy
    this.selectStrategy = ObjectUtil.checkNotNull(strategy, "selectStrategy");
    // **************************************************
    // 7.打开选择器(并为选择器配置自定义的结果队列)
    // **************************************************
    final SelectorTuple selectorTuple = openSelector();
    
    // Netty自定义的的Selector,内部包含有原生:Selector
    // selector = io.netty.channel.nio.SelectedSelectionKeySetSelector
    this.selector = selectorTuple.selector;
    
    // 未经包裹的JDK原生:Selector
    // unwrappedSelector = sun.nio.ch.KQueueSelectorImpl
    this.unwrappedSelector = selectorTuple.unwrappedSelector;
}

private static Queue<Runnable> newTaskQueue(
            EventLoopTaskQueueFactory queueFactory) {
    // queueFactory = null
    if (queueFactory == null) { // true
        // DEFAULT_MAX_PENDING_TASKS = 2147483647
        return newTaskQueue0(DEFAULT_MAX_PENDING_TASKS);
    }
    return queueFactory.newTaskQueue(DEFAULT_MAX_PENDING_TASKS);
}

 private static Queue<Runnable> newTaskQueue0(int maxPendingTasks) {
    // maxPendingTasks = 2147483647
    // true
    return maxPendingTasks == Integer.MAX_VALUE 
                ? PlatformDependent.<Runnable>newMpscQueue()
                : PlatformDependent.<Runnable>newMpscQueue(maxPendingTasks);
}
```

### (3).SingleThreadEventLoop构造器
```
protected SingleThreadEventLoop(EventLoopGroup parent, Executor executor,
                                    boolean addTaskWakesUp, Queue<Runnable> taskQueue, Queue<Runnable> tailTaskQueue,
                                    RejectedExecutionHandler rejectedExecutionHandler) {
    // 调用父类(SingleThreadEventExecutor)构造器
    super(parent, executor, addTaskWakesUp, taskQueue, rejectedExecutionHandler);
    // tailTasks = org.jctools.queues.MpscUnboundedArrayQueue
    tailTasks = ObjectUtil.checkNotNull(tailTaskQueue, "tailTaskQueue");
}
```

### (4).SingleThreadEventExecutor构造器
```
protected SingleThreadEventExecutor(
        EventExecutorGroup parent, 
        Executor executor,
        boolean addTaskWakesUp, 
        Queue<Runnable> taskQueue,
        RejectedExecutionHandler rejectedHandler) {
            
    // 调用父类(AbstractScheduledEventExecutor)构造器
    super(parent);
    // addTaskWakesUp = false
    this.addTaskWakesUp = addTaskWakesUp;
    // maxPendingTasks = 2147483647
    this.maxPendingTasks = DEFAULT_MAX_PENDING_EXECUTOR_TASKS;
    // executor = io.netty.util.internal.ThreadExecutorMap$1
   this.executor = ThreadExecutorMap.apply(executor, this);
   // taskQueue = org.jctools.queues.MpscUnboundedArrayQueue
   this.taskQueue = ObjectUtil.checkNotNull(taskQueue, "taskQueue");
   // rejectedExecutionHandler = io.netty.util.concurrent.RejectedExecutionHandlers$1
   this.rejectedExecutionHandler = ObjectUtil.checkNotNull(rejectedHandler, "rejectedHandler");
}
```

### (5).AbstractScheduledEventExecutor构造器
```
protected AbstractScheduledEventExecutor(EventExecutorGroup parent) {
    // 调用父类(AbstractEventExecutor)构造器
    super(parent);
}
```

### (6).AbstractEventExecutor构造器
```
protected AbstractEventExecutor(EventExecutorGroup parent) {
    this.parent = parent;
}
```

### (7).NioEventLoop.openSelector
```
private SelectorTuple openSelector() {
        final Selector unwrappedSelector;
        try {
            //unwrappedSelector =  sun.nio.ch.KQueueSelectorImpl
            unwrappedSelector = provider.openSelector();
        } catch (IOException e) {
            throw new ChannelException("failed to open a new selector", e);
        }

        if (DISABLE_KEY_SET_OPTIMIZATION) { // false
            return new SelectorTuple(unwrappedSelector);
        }

         // 获取class:sun.nio.ch.SelectorImpl的信息
        // maybeSelectorImplClass = sun.nio.ch.SelectorImpl
        Object maybeSelectorImplClass = AccessController.doPrivileged(new PrivilegedAction<Object>() {
            @Override
            public Object run() {
                try {
                    return Class.forName(
                            "sun.nio.ch.SelectorImpl",
                            false,
                            PlatformDependent.getSystemClassLoader());
                } catch (Throwable cause) {
                    return cause;
                }
            }
        });

        // false
        if (!(maybeSelectorImplClass instanceof Class) ||
            // ensure the current selector implementation is what we can instrument.
            !((Class<?>) maybeSelectorImplClass).isAssignableFrom(unwrappedSelector.getClass())) {
            if (maybeSelectorImplClass instanceof Throwable) {
                Throwable t = (Throwable) maybeSelectorImplClass;
                logger.trace("failed to instrument a special java.util.Set into: {}", unwrappedSelector, t);
            }
            return new SelectorTuple(unwrappedSelector);
        }

        // selectorImplClass = sun.nio.ch.SelectorImpl
        final Class<?> selectorImplClass = (Class<?>) maybeSelectorImplClass;
        final SelectedSelectionKeySet selectedKeySet = new SelectedSelectionKeySet();

        
        // *************************************************
        // 为实例对象(sun.nio.ch.KQueueSelectorImpl)
        // 设置:selectedKeys/publicSelectedKeys属性指定为:自定义的结果存储集合(SelectedSelectionKeySet)里.
        // *************************************************
        Object maybeException = AccessController.doPrivileged(new PrivilegedAction<Object>() {
            @Override
            public Object run() {
                try {
                    Field selectedKeysField = selectorImplClass.getDeclaredField("selectedKeys");
                    Field publicSelectedKeysField = selectorImplClass.getDeclaredField("publicSelectedKeys");

                    if (PlatformDependent.javaVersion() >= 9 && PlatformDependent.hasUnsafe()) {
                        // Let us try to use sun.misc.Unsafe to replace the SelectionKeySet.
                        // This allows us to also do this in Java9+ without any extra flags.
                        long selectedKeysFieldOffset = PlatformDependent.objectFieldOffset(selectedKeysField);
                        long publicSelectedKeysFieldOffset =
                                PlatformDependent.objectFieldOffset(publicSelectedKeysField);

                        if (selectedKeysFieldOffset != -1 && publicSelectedKeysFieldOffset != -1) {
                            PlatformDependent.putObject(
                                    unwrappedSelector, selectedKeysFieldOffset, selectedKeySet);
                            PlatformDependent.putObject(
                                    unwrappedSelector, publicSelectedKeysFieldOffset, selectedKeySet);
                            return null;
                        }
                        // We could not retrieve the offset, lets try reflection as last-resort.
                    }

                    Throwable cause = ReflectionUtil.trySetAccessible(selectedKeysField, true);
                    if (cause != null) {
                        return cause;
                    }
                    cause = ReflectionUtil.trySetAccessible(publicSelectedKeysField, true);
                    if (cause != null) {
                        return cause;
                    }
                    
                    
                    // unwrappedSelector = sun.nio.ch.KQueueSelectorImpl
                    // sun.nio.ch.KQueueSelectorImpl.selectedKeys
                    // sun.nio.ch.KQueueSelectorImpl.publicSelectedKeys
                    selectedKeysField.set(unwrappedSelector, selectedKeySet);
                    publicSelectedKeysField.set(unwrappedSelector, selectedKeySet);
                    return null;
                } catch (NoSuchFieldException e) {
                    return e;
                } catch (IllegalAccessException e) {
                    return e;
                }
            }
        });

        if (maybeException instanceof Exception) { // false
            selectedKeys = null;
            Exception e = (Exception) maybeException;
            logger.trace("failed to instrument a special java.util.Set into: {}", unwrappedSelector, e);
            return new SelectorTuple(unwrappedSelector);
        }

        // ****************************************************
        // SelectedSelectionKeySet 内部持有:多人java.nio.channels.SelectionKey
        // ****************************************************
        selectedKeys = selectedKeySet;
        
        logger.trace("instrumented a special java.util.Set into: {}", unwrappedSelector);
        return new SelectorTuple(
            // 未包裹的Selector
            // sun.nio.ch.KQueueSelectorImpl
            unwrappedSelector,
            // selectedKeys=[]
            // SelectedSelectionKeySetSelector 是java.nio.channels.Selector的实现类
            new SelectedSelectionKeySetSelector(unwrappedSelector, selectedKeySet)
       ); //end new SelectorTuple
    }
    

// class SelectedSelectionKeySetSelector extends Selector {}
// class SelectedSelectionKeySet extends AbstractSet<SelectionKey> {}

```

### (8).SelectedSelectionKeySetSelector
```
// 该类会委托给:KQueueSelectorImpl进行处理
final class SelectedSelectionKeySetSelector extends Selector {
    private final SelectedSelectionKeySet selectionKeys;
    private final Selector delegate;

    SelectedSelectionKeySetSelector(
            Selector delegate, 
            SelectedSelectionKeySet selectionKeys) {
        // delegate = sun.nio.ch.KQueueSelectorImpl
        this.delegate = delegate;
       // selectionKeys = io.netty.channel.nio.SelectedSelectionKeySet
       this.selectionKeys = selectionKeys;
    }

    @Override
    public boolean isOpen() {
        return delegate.isOpen();
    }

    @Override
    public SelectorProvider provider() {
        return delegate.provider();
    }

    @Override
    public Set<SelectionKey> keys() {
        return delegate.keys();
    }

    @Override
    public Set<SelectionKey> selectedKeys() {
        return delegate.selectedKeys();
    }

    @Override
    public int selectNow() throws IOException {
        selectionKeys.reset();
        return delegate.selectNow();
    }

    @Override
    public int select(long timeout) throws IOException {
        selectionKeys.reset();
        return delegate.select(timeout);
    }

    @Override
    public int select() throws IOException {
        selectionKeys.reset();
        return delegate.select();
    }

    @Override
    public Selector wakeup() {
        return delegate.wakeup();
    }

    @Override
    public void close() throws IOException {
        delegate.close();
    }
}

```
### (9).总结
> NioEventLoop构造时,会创建:JDK 的Selector以及无界队列.