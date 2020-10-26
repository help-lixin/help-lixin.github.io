---
layout: post
title: 'Netty源码学习(ServerBootstrap)'
date: 2020-10-26
author: 李新
tags: Netty
---

### (1).ServerBootstrap继承关系
```
io.netty.bootstrap.ServerBootstrap 
    extends io.netty.bootstrap.AbstractBootstrap

// ServerBootstrap只是继承了:AbstractBootstrap
// AbstractBootstrap 主要是针对BossGroup的一些函数定义
// ServerBootstrap   主要是针对WorkGroup的一些函数定义
```

### (2).ServerBootstrap创建与使用
```
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
EventLoopGroup workerGroup = new NioEventLoopGroup();
try {
    // 2.创建ServerBootstrap
    ServerBootstrap b = new ServerBootstrap();
    // 3.进行配置
    // group配置
    b.group(bossGroup, workerGroup)
     .channel(NioServerSocketChannel.class)
     .handler(new LoggingHandler(LogLevel.INFO))
    .childHandler(new TelnetServerInitializer(sslCtx));

    b.bind(PORT).sync().channel().closeFuture().sync();
} finally {
    bossGroup.shutdownGracefully();
    workerGroup.shutdownGracefully();
}
```

### (3).创建ServerBootstrap
```
// 无参构造器
public ServerBootstrap() { }

```

### (4).ServerBootstrap.group配置
```
public ServerBootstrap group(EventLoopGroup group) {
    return group(group, group);
}

public ServerBootstrap group(EventLoopGroup parentGroup, EventLoopGroup childGroup) {
    // 5. 调用父类:AbstractBootstrap构造器
    super.group(parentGroup);
    
    if (this.childGroup != null) {
        throw new IllegalStateException("childGroup set already");
    }
    // 构建childGroup
    this.childGroup = ObjectUtil.checkNotNull(childGroup, "childGroup");
    return this;
}
```

### (5).AbstractBootstrap构造器
```
volatile EventLoopGroup group;

public B group(EventLoopGroup group) {
    ObjectUtil.checkNotNull(group, "group");
    if (this.group != null) {
        throw new IllegalStateException("group set already");
    }
    this.group = group;
    return self();
}
```

### (6).AbstractBootstrap.channel配置
```
public B channel(Class<? extends C> channelClass) {
    // channelClass = io.netty.channel.socket.nio.NioServerSocketChannel
    // 8.AbstractBootstrap.channelFactory
    return channelFactory(
        // 7.创建ReflectiveChannelFactory
        new ReflectiveChannelFactory<C>(ObjectUtil.checkNotNull(channelClass, "channelClass"))
    );
}
```

### (7).ReflectiveChannelFactory 构造器
```
public class ReflectiveChannelFactory<T extends Channel> implements ChannelFactory<T> {
    private final Constructor<? extends T> constructor;

    public ReflectiveChannelFactory(Class<? extends T> clazz) {
        ObjectUtil.checkNotNull(clazz, "clazz");
        try {
            this.constructor = clazz.getConstructor();
        } catch (NoSuchMethodException e) {
            throw new IllegalArgumentException("Class " + StringUtil.simpleClassName(clazz) +
                    " does not have a public non-arg constructor", e);
        }
    } //end 构造器
    
    public T newChannel() {
        try {
            return constructor.newInstance();
        } catch (Throwable t) {
            throw new ChannelException("Unable to create Channel from class " + constructor.getDeclaringClass(), t);
        }
    } // end newChannel
}
```

### (8).AbstractBootstrap.channelFactory
```
public B channelFactory(ChannelFactory<? extends C> channelFactory) {
    // channelFactory = ReflectiveChannelFactory(NioServerSocketChannel.class)
    ObjectUtil.checkNotNull(channelFactory, "channelFactory");
    
    if (this.channelFactory != null) { // false
        throw new IllegalStateException("channelFactory set already");
    }
    // 设置Channel工厂
    this.channelFactory = channelFactory;
    return self();
}
```

### (9).AbstractBootstrap.handler 配置日志
```
// 配置bossHadnler
private volatile ChannelHandler handler;

public B handler(ChannelHandler handler) {
    // handler = io.netty.handler.logging.LoggingHandler
    this.handler = ObjectUtil.checkNotNull(handler, "handler");
    return self();
}
```

### (10).ServerBootstrap.childHandler
```
// 配置workHandler

private volatile ChannelHandler childHandler;

public ServerBootstrap childHandler(ChannelHandler childHandler) {
    // childHandler = io.netty.example.telnet.TelnetServerInitializer
    this.childHandler = ObjectUtil.checkNotNull(childHandler, "childHandler");
    return this;
}

```

### (11).AbstractBootstrap.bind
```
public ChannelFuture bind(int inetPort) {
    // inetPort = 8023
    return bind(new InetSocketAddress(inetPort));
}

public ChannelFuture bind(SocketAddress localAddress) {
    // localAddress = 0.0.0.0/0.0.0.0:8023
    
    // 验证:channelFactory/group都不能为空
    validate();
    
    // 调用doBind
    return doBind(ObjectUtil.checkNotNull(localAddress, "localAddress"));
}

private ChannelFuture doBind(final SocketAddress localAddress) {
    // 12. 初始化并注册
    final ChannelFuture regFuture = initAndRegister();
    final Channel channel = regFuture.channel();
    if (regFuture.cause() != null) {
            return regFuture;
    } //end if
    
    if (regFuture.isDone()) {
        // At this point we know that the registration was complete and successful.
        ChannelPromise promise = channel.newPromise();
        doBind0(regFuture, channel, localAddress, promise);
        return promise;
    } else {
        // Registration future is almost always fulfilled already, but just in case it's not.
        final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
        regFuture.addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    Throwable cause = future.cause();
                    if (cause != null) {
                        // Registration on the EventLoop failed so fail the ChannelPromise directly to not cause an
                        // IllegalStateException once we try to access the EventLoop of the Channel.
                        promise.setFailure(cause);
                    } else {
                        // Registration was successful, so set the correct executor to use.
                        // See https://github.com/netty/netty/issues/2586
                        promise.registered();
                        doBind0(regFuture, channel, localAddress, promise);
                    }
                }
            }); //end addListener
            return promise;
    } //end else
}
```

### (12).AbstractBootstrap.initAndRegister
```
// 初始化:ChannelFuture
final ChannelFuture initAndRegister() {
    Channel channel = null;
    try {
        // *****************************************************************
        // 通过反射工厂创建出:io.netty.channel.socket.nio.NioServerSocketChannel
        // NioServerSocketChannel另起一遍进行分析
        // *****************************************************************
        channel = channelFactory.newChannel();
        // 13. 调用子类(ServerBootstrap),配置:NioServerSocketChannel
        init(channel);
    } catch (Throwable t) {
        if (channel != null) {
            // channel can be null if newChannel crashed (eg SocketException("too many open files"))
            channel.unsafe().closeForcibly();
            // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
            return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
        }
        // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
        return new DefaultChannelPromise(new FailedChannel(), GlobalEventExecutor.INSTANCE).setFailure(t);
    }
    
    // ****************************************************
    // ****************************************************
    // config().group() == NioEventLoopGroup
    // 往BossGroup(NioEventLoopGroup)中注册:NioServerSocketChannel
    // ****************************************************
    // ****************************************************
    ChannelFuture regFuture = config().group().register(channel);
    
    if (regFuture.cause() != null) {  // false
        if (channel.isRegistered()) {
            channel.close();
        } else {
            channel.unsafe().closeForcibly();
        }
    }
    return regFuture;
} // end initAndRegister
```

### (13).ServerBootstrap.init
```
// 对NioServerSocketChannel进行相关配置
void init(Channel channel) {
    // channel = NioServerSocketChannel
    setChannelOptions(channel, newOptionsArray(), logger);
    setAttributes(channel, attrs0().entrySet().toArray(EMPTY_ATTRIBUTE_ARRAY));

    // DefaultChannelPipeline
    ChannelPipeline p = channel.pipeline();

    // io.netty.channel.nio.NioEventLoopGroup
    // 获得workGroup
   final EventLoopGroup currentChildGroup = childGroup;
   // 获得workgChild
   // io.netty.example.telnet.TelnetServerInitializer
   final ChannelHandler currentChildHandler = childHandler;
   
   // child配置项
   final Entry<ChannelOption<?>, Object>[] currentChildOptions;
    synchronized (childOptions) {
        currentChildOptions = childOptions.entrySet().toArray(EMPTY_OPTION_ARRAY);
    }
    
    // []
    final Entry<AttributeKey<?>, Object>[] currentChildAttrs = childAttrs.entrySet().toArray(EMPTY_ATTRIBUTE_ARRAY);
    
    // **************************************************
    // 在默认的DefaultChannelPipeline的基础上添加一个:io.netty.channel.ChannelHandler
    // **************************************************
    p.addLast(new ChannelInitializer<Channel>() {
        @Override
        public void initChannel(final Channel ch) {
            final ChannelPipeline pipeline = ch.pipeline();
            ChannelHandler handler = config.handler();
            if (handler != null) {
                pipeline.addLast(handler);
            }
            
            // ****************************************
            // 添加任务到队列里
            // ****************************************
            ch.eventLoop().execute(new Runnable() {
                @Override
                public void run() {
                    pipeline.addLast(new ServerBootstrapAcceptor(
                            ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                }
            });
        }
    }); // end addLast
}
```

### (14).MultithreadEventLoopGroup(NioEventLoopGroup).register
```
public ChannelFuture register(Channel channel) {
    // channel = NioServerSocketChannel
    // next() --> io.netty.util.concurrent.DefaultEventExecutorChooserFactory$PowerOfTwoEventExecutorChooser
    // next() 方法相当于从多个EventExecutor(NioEventLoop)中选择一个,类似于策略模式
    // next() 返回的类型为:EventExecutor(NioEventLoop)
    return next().register(channel);
}
```

### (15).SingleThreadEventLoop(NioEventLoop).register
```
public ChannelFuture register(Channel channel) {
    // channel = NioServerSocketChannel
    return register(new DefaultChannelPromise(channel, this));
}

public ChannelFuture register(final ChannelPromise promise) {
    ObjectUtil.checkNotNull(promise, "promise");
    // channel() == NioServerSocketChannel
    // ************************************************************************************************************
   // NioServerSocketChannel的构造器在初始化时,会调用父类的构造器:AbstractChannel
   // 而父类构造器会调用:AbstractNioMessageChannel.newUnsafe() 构建出一个:AbstractNioMessageChannel$NioMessageUnsafe
   // ************************************************************************************************************
   
    promise.channel().unsafe().register(this, promise);
    return promise;
}
```

### (16).AbstractNioMessageChannel.NioMessageUnsafe.register
```
// ********************************************************************************************
// AbstractNioMessageChannel.NioMessageUnsafe 继承于:AbstractChannel.AbstractUnsafe
// ********************************************************************************************

public abstract class AbstractChannel extends DefaultAttributeMap implements Channel {
    
    protected abstract class AbstractUnsafe implements Unsafe {
        
        public final void register(EventLoop eventLoop, final ChannelPromise promise) {
            // eventLoop = io.netty.channel.nio.NioEventLoop
            // promise = DefaultChannelPromise
            ObjectUtil.checkNotNull(eventLoop, "eventLoop");
            
            if (isRegistered()) { // false
                promise.setFailure(new IllegalStateException("registered to an event loop already"));
                return;
            }
            
            if (!isCompatible(eventLoop)) { // false
                promise.setFailure(
                        new IllegalStateException("incompatible event loop type: " + eventLoop.getClass().getName()));
                return;
            }

            // 这个方法应该只可能调用一次
            // 给eventLoop赋值
            AbstractChannel.this.eventLoop = eventLoop;

            // 判断:NioEventLoop中的线程是否为当前线程
            // 这个时候,NioEventLoop中的thread的属性还为空中.
            if (eventLoop.inEventLoop()) { // false
                register0(promise);
            } else {
                try {
                 // ***************************************************
                // 调用:SingleThreadEventExecutor(NioEventLoop).execute(....)
                // ***************************************************
                    eventLoop.execute(new Runnable() {
                        @Override
                        public void run() {
                            register0(promise);
                        }
                    });
                //     
                } catch (Throwable t) {
                    logger.warn(
                            "Force-closing a channel whose registration task was not accepted by an event loop: {}",
                            AbstractChannel.this, t);
                    closeForcibly();
                    closeFuture.setClosed();
                    safeSetFailure(promise, t);
                }
            }// 
        } // end register
    }
}

```
### (17).SingleThreadEventExecutor(NioEventLoop).execute
```
public void execute(Runnable task) {
    // task = io.netty.channel.AbstractChannel$AbstractUnsafe$1
    ObjectUtil.checkNotNull(task, "task");
    execute(task, !(task instanceof LazyRunnable) && wakesUpForTask(task));
}
```

### (18).SingleThreadEventExecutor(NioEventLoop).execute
```
private void execute(Runnable task, boolean immediate) {
    // task = io.netty.channel.AbstractChannel$AbstractUnsafe.register0
    // immediate = true
    // inEventLoop = false
    boolean inEventLoop = inEventLoop();
    // 添加任务进入队列
    addTask(task);
    if (!inEventLoop) { //!(false)
        // 18.开启线程
        startThread();
        
        if (isShutdown()) {
            boolean reject = false;
            try {
                if (removeTask(task)) {
                    reject = true;
                } // end if
            } catch (UnsupportedOperationException e) {
            }
            
            if (reject) {
                reject();
            } // end if
            
        } // end if
    } // end if

    if (!addTaskWakesUp && immediate) {
        wakeup(inEventLoop);
    } // end if
}



protected void addTask(Runnable task) {
    ObjectUtil.checkNotNull(task, "task");
    if (!offerTask(task)) { // !(true)
        reject(task);
    }
} //end addTask


// 线程状态
volatile int state = ST_NOT_STARTED;
// 任务队列
final Queue<Runnable> taskQueue;

final boolean offerTask(Runnable task) {
    // 如果线程的状态是关闭,则拒绝,否则:添加到任务列表中
    if (isShutdown()) { // false
        reject();
    }
    // 添加到任务队列里
    return taskQueue.offer(task);
}
```

### (19).SingleThreadEventExecutor(NioEventLoop).startThread
```
private void startThread() {
    // ST_NOT_STARTED == 未启动
    // ST_STARTED == 启动
    // ST_SHUTTING_DOWN == 关闭完成
    // ST_SHUTDOWN == 关闭
    // ST_TERMINATED == 中止
    // state 初始化时的默认值是:ST_NOT_STARTED
    if (state == ST_NOT_STARTED) { // true
        // 状态更新器
        // 比较NioEventLoop的state
        // 如果state = ST_NOT_STARTED 则更新为:ST_STARTED
        if (STATE_UPDATER.compareAndSet(this, ST_NOT_STARTED, ST_STARTED)) { // true
            boolean success = false;
            try {
                // 开始启动线程.
                doStartThread();
                success = true;
            } finally {
                // 如果没有成功,则重新设置state == ST_NOT_STARTED(未启动)
                if (!success) {
                    STATE_UPDATER.compareAndSet(this, ST_STARTED, ST_NOT_STARTED);
                }
            } // end finally
        } // end if
    }// end if
}
```

### (20).SingleThreadEventExecutor(NioEventLoop).doStartThread
```
private void doStartThread() {
    assert thread == null;
    // ******************************************************************
    // 另启动一个线程去初始化Thread 并调用:SingleThreadEventExecutor.run方法
    // ******************************************************************
    executor.execute(new Runnable() {
        @Override
        public void run() {
            // thread = Thread[nioEventLoopGroup-2-1,10,main]
            thread = Thread.currentThread();
            
            if (interrupted) { // false
                thread.interrupt();
            }

            boolean success = false;
            // 更新最后执行线程的时间
            updateLastExecutionTime();
            try {
                // *********************************************************
                // 调用:NioEventLoop.run()方法
                // *********************************************************
                SingleThreadEventExecutor.this.run();
                success = true;
            } catch (Throwable t) {
                logger.warn("Unexpected exception from an event executor: ", t);
            } finally {
                for (;;) {
                    int oldState = state;
                    if (oldState >= ST_SHUTTING_DOWN || STATE_UPDATER.compareAndSet(
                            SingleThreadEventExecutor.this, oldState, ST_SHUTTING_DOWN)) {
                        break;
                    }
                }

                // Check if confirmShutdown() was called at the end of the loop.
                if (success && gracefulShutdownStartTime == 0) {
                    if (logger.isErrorEnabled()) {
                        logger.error("Buggy " + EventExecutor.class.getSimpleName() + " implementation; " +
                                SingleThreadEventExecutor.class.getSimpleName() + ".confirmShutdown() must " +
                                "be called before run() implementation terminates.");
                    }
                }

                try {
                    // Run all remaining tasks and shutdown hooks. At this point the event loop
                    // is in ST_SHUTTING_DOWN state still accepting tasks which is needed for
                    // graceful shutdown with quietPeriod.
                    for (;;) {
                        if (confirmShutdown()) {
                            break;
                        }
                    }

                    // Now we want to make sure no more tasks can be added from this point. This is
                    // achieved by switching the state. Any new tasks beyond this point will be rejected.
                    for (;;) {
                        int oldState = state;
                        if (oldState >= ST_SHUTDOWN || STATE_UPDATER.compareAndSet(
                                SingleThreadEventExecutor.this, oldState, ST_SHUTDOWN)) {
                            break;
                        }
                    }

                    // We have the final set of tasks in the queue now, no more can be added, run all remaining.
                    // No need to loop here, this is the final pass.
                    confirmShutdown();
                } finally {
                    try {
                        cleanup();
                    } finally {
                        // Lets remove all FastThreadLocals for the Thread as we are about to terminate and notify
                        // the future. The user may block on the future and once it unblocks the JVM may terminate
                        // and start unloading classes.
                        // See https://github.com/netty/netty/issues/6596.
                        FastThreadLocal.removeAll();

                        STATE_UPDATER.set(SingleThreadEventExecutor.this, ST_TERMINATED);
                        threadLock.countDown();
                        int numUserTasks = drainTasks();
                        if (numUserTasks > 0 && logger.isWarnEnabled()) {
                            logger.warn("An event executor terminated with " +
                                    "non-empty task queue (" + numUserTasks + ')');
                        }
                        terminationFuture.setSuccess(null);
                    }
                }
            }
        }
    });
} // end doStartThread

```

### (21).NioEventLoop.run
```
protected void run() {
    int selectCnt = 0;
    for (;;) {  // 死循环
        try {
            int strategy;
            try {
                // 判断是否有事件或者队列是否有数据
                strategy = selectStrategy.calculateStrategy(selectNowSupplier, hasTasks());
                // strategy = 0
                // 整个switch都不进入
                switch (strategy) { 
                case SelectStrategy.CONTINUE:
                    continue;

                case SelectStrategy.BUSY_WAIT:
                    // fall-through to SELECT since the busy-wait is not supported with NIO

                case SelectStrategy.SELECT:
                    long curDeadlineNanos = nextScheduledTaskDeadlineNanos();
                    if (curDeadlineNanos == -1L) {
                        curDeadlineNanos = NONE; // nothing on the calendar
                    }
                    nextWakeupNanos.set(curDeadlineNanos);
                    try {
                        if (!hasTasks()) {
                            strategy = select(curDeadlineNanos);
                        }
                    } finally {
                        // This update is just to help block unnecessary selector wakeups
                        // so use of lazySet is ok (no race condition)
                        nextWakeupNanos.lazySet(AWAKE);
                    }
                    // fall through
                default:
                }
            } catch (IOException e) {
                // If we receive an IOException here its because the Selector is messed up. Let's rebuild
                // the selector and retry. https://github.com/netty/netty/issues/8566
                rebuildSelector0();
                selectCnt = 0;
                handleLoopException(e);
                continue;
            }
            
            // 统计值进行自增
            selectCnt++;
            cancelledKeys = 0;
            needsToSelectAgain = false;
            final int ioRatio = this.ioRatio;
            boolean ranTasks;
            // ioRatio = 50 
            if (ioRatio == 100) { // false
                try {
                    if (strategy > 0) {
                        processSelectedKeys();
                    }
                } finally {
                    // Ensure we always run tasks.
                    ranTasks = runAllTasks();
                }
            } else if (strategy > 0) { // false(strategy == 0)
                final long ioStartTime = System.nanoTime();
                try {
                    processSelectedKeys();
                } finally {
                    // Ensure we always run tasks.
                    final long ioTime = System.nanoTime() - ioStartTime;
                    ranTasks = runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                }
            } else {
                // ************************************************
                // 运行所有的任务
                // ************************************************
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
        }
    }
} // end run
```

### (22).SingleThreadEventExecutor(NioEventLoop).runAllTasks
```
protected boolean runAllTasks(long timeoutNanos) {
    // timeoutNanos == 0
    fetchFromScheduledTaskQueue();
    // 从队列里拉取任务
    Runnable task = pollTask();
    
    // 如果任务不为空,则返回
    if (task == null) { // false
        afterRunningAllTasks();
        return false;
    }

    // deadline = 0
    final long deadline = timeoutNanos > 0 ? ScheduledFutureTask.nanoTime() + timeoutNanos : 0;
    long runTasks = 0;
    long lastExecutionTime;
    
    
    for (;;) {
        // ********************************************************
        // 此时会回调:AbstractChannel$AbstractUnsafe.register0()
        //          ServerBootstrap$init
        // ********************************************************
        safeExecute(task);

        runTasks ++;
        
        if ((runTasks & 0x3F) == 0) {
            lastExecutionTime = ScheduledFutureTask.nanoTime();
            if (lastExecutionTime >= deadline) {
                break;
            }
        }

        // 如果队列里一个有内容,那这个run是不是代表不会退出
        // 再从队列里取出一个任务出来
        // ServerBootstrap$init
        // ServerBootstrapAcceptor
        task = pollTask();
        if (task == null) {
            lastExecutionTime = ScheduledFutureTask.nanoTime();
            break;
        }
    }

    afterRunningAllTasks();
    this.lastExecutionTime = lastExecutionTime;
    return true;
} // end runAllTasks


private boolean fetchFromScheduledTaskQueue() {
    // scheduledTaskQueue = null
    if (scheduledTaskQueue == null || scheduledTaskQueue.isEmpty()) {
        // 直接返回了
        return true;
    }
    long nanoTime = AbstractScheduledEventExecutor.nanoTime();
    for (;;) {
        Runnable scheduledTask = pollScheduledTask(nanoTime);
        if (scheduledTask == null) {
            return true;
        }
        if (!taskQueue.offer(scheduledTask)) {
            // No space left in the task queue add it back to the scheduledTaskQueue so we pick it up again.
            scheduledTaskQueue.add((ScheduledFutureTask<?>) scheduledTask);
            return false;
        }
    }
} // end fetchFromScheduledTaskQueue


protected Runnable pollTask() {
    assert inEventLoop();
    return pollTaskFrom(taskQueue);
}

protected static Runnable pollTaskFrom(Queue<Runnable> taskQueue) {
    for (;;) {
        // 从队列里拉取出数
        Runnable task = taskQueue.poll();
        // 如果任务不是唤醒任务,则返回任务
        if (task != WAKEUP_TASK) {
            return task;
        }
    }
}

```

### (23).AbstractChannel(NioServerSocketChannel)$AbstractUnsafe.register0
```
private void register0(ChannelPromise promise) {
    try {
        
        if (!promise.setUncancellable() || !ensureOpen(promise)) { // false
            return;
        }
        
        // firstRegistration
        boolean firstRegistration = neverRegistered;
       //  **********************************************************************
       // AbstractNioChannel$AbstractNioUnsafe.doRegister()
       // 调用子类的doRegister
       // 把Channel与Selector进行绑定
       //  **********************************************************************
       doRegister();
       neverRegistered = false;
       registered = true;
       
       // pipeline = io.netty.channel.DefaultChannelPipeline 
       // **********************************************************
       // 24.调用所有的:ChanelHandler.handlerAdded
       // **********************************************************
       pipeline.invokeHandlerAddedIfNeeded();


       safeSetSuccess(promise);
       
       // ****************************************
       // 28. DefaultChannelPipeline.fireChannelRegistered
       // 调用所有ChannelHandlerContext对应handler.channelRegistered
       // ****************************************
       pipeline.fireChannelRegistered();
       
       if (isActive()) { // false
           if (firstRegistration) {
                pipeline.fireChannelActive();
           } else if (config().isAutoRead()) {
                // This channel was registered before and autoRead() is set. This means we need to begin read
                // again so that we process inbound data.
                //
                // See https://github.com/netty/netty/issues/4805
                beginRead();
           } // end else if
       } // end if
    } catch (Throwable t) {
        // Close the channel directly to avoid FD leak.
        closeForcibly();
        closeFuture.setClosed();
        safeSetFailure(promise, t);
    }
} // end register0

```

### (24).AbstractNioChannel(NioServerSocketChannel)$AbstractNioUnsafe.doRegister()
```
protected void doRegister() throws Exception {
    boolean selected = false;
    for (;;) {
        try {
            // **************************************************
            // 调用JDK原生的注册事件,将:Channel与Selector进行绑定
            // eventLoop().unwrappedSelector() = sun.nio.ch.KQueueSelectorImpl
            // javaChannel() = sun.nio.ch.ServerSocketChannelImpl
            // **************************************************
            selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
            return;
        } catch (CancelledKeyException e) {
            if (!selected) {
                // Force the Selector to select now as the "canceled" SelectionKey may still be
                // cached and not removed because no Select.select(..) operation was called yet.
                eventLoop().selectNow();
                selected = true;
            } else {
                // We forced a select operation on the selector before but the SelectionKey is still cached
                // for whatever reason. JDK bug ?
                throw e;
            }
        }
    } // end for
} // end doRegister

```

### (25).DefaultChannelPipeline.invokeHandlerAddedIfNeeded
```
final void invokeHandlerAddedIfNeeded() {
    assert channel.eventLoop().inEventLoop();
    // 第一次注册
    if (firstRegistration) { // true
        firstRegistration = false;
        // 
        callHandlerAddedForAllHandlers();
    }
} // end invokeHandlerAddedIfNeeded


private void callHandlerAddedForAllHandlers() {

    final PendingHandlerCallback pendingHandlerCallbackHead;
    synchronized (this) {
        assert !registered;
        registered = true;
        // 1. 获得实例变量
        // this.pendingHandlerCallbackHead = io.netty.channel.DefaultChannelPipeline$PendingHandlerAddedTask
        // pendingHandlerCallbackHead = io.netty.channel.DefaultChannelPipeline$PendingHandlerAddedTask
        pendingHandlerCallbackHead = this.pendingHandlerCallbackHead;
        
        // 2. 清空实例变量的值.
        this.pendingHandlerCallbackHead = null;
    }
    
    // task = io.netty.channel.DefaultChannelPipeline$PendingHandlerAddedTask
    PendingHandlerCallback task = pendingHandlerCallbackHead;
    while (task != null) {
        // 执行execute()方法
        task.execute();
        task = task.next;
    }
} // end callHandlerAddedForAllHandlers
```

### (26).PendingHandlerAddedTask.execute
```
void execute() {
    // 从上下文中获得:executeor
    // executor = io.netty.channel.nio.NioEventLoop
    EventExecutor executor = ctx.executor();
    
    if (executor.inEventLoop()) { // true
        // *******************************************
        // 调用handlerAdd方法
        // *******************************************
        callHandlerAdded0(ctx);
    } else {
        try {
            executor.execute(this);
        } catch (RejectedExecutionException e) {
            if (logger.isWarnEnabled()) {
                logger.warn(
                        "Can't invoke handlerAdded() as the EventExecutor {} rejected it, removing handler {}.",
                        executor, ctx.name(), e);
            }
            atomicRemoveFromHandlerList(ctx);
            ctx.setRemoved();
        }
    }
} // end execute
```
### (27).DefaultChannelPipeline.callHandlerAdded0
```
private void callHandlerAdded0(final AbstractChannelHandlerContext ctx) {
    try {
        // 调用handlerAdded方法
        ctx.callHandlerAdded();
    } catch (Throwable t) {
        boolean removed = false;
        try {
            atomicRemoveFromHandlerList(ctx);
            ctx.callHandlerRemoved();
            removed = true;
        } catch (Throwable t2) {
            if (logger.isWarnEnabled()) {
                logger.warn("Failed to remove a handler: " + ctx.name(), t2);
            }
        }
    
        if (removed) {
            fireExceptionCaught(new ChannelPipelineException(
                    ctx.handler().getClass().getName() +
                    ".handlerAdded() has thrown an exception; removed.", t));
        } else {
            fireExceptionCaught(new ChannelPipelineException(
                    ctx.handler().getClass().getName() +
                    ".handlerAdded() has thrown an exception; also failed to remove.", t));
        }
    } // end catch
} // end callHandlerAdded0

```

### (28).DefaultChannelHandlerContext.callHandlerAdded
```
final void callHandlerAdded() throws Exception {
    if (setAddComplete()) { //true
        //  handler为业务的ChannelHandler
        // io.netty.bootstrap.ServerBootstrap$1
        handler().handlerAdded(this);
    }
}

final boolean setAddComplete() {
    for (;;) {
        int oldState = handlerState;
        if (oldState == REMOVE_COMPLETE) {
            return false;
        }
        // 循环修改状态为:ADD_COMPLETE
        if (HANDLER_STATE_UPDATER.compareAndSet(this, oldState, ADD_COMPLETE)) {
            return true;
        }
    } // end 
}

```
### (29).DefaultChannelPipeline.fireChannelRegistered
```
public final ChannelPipeline fireChannelRegistered() {
    AbstractChannelHandlerContext.invokeChannelRegistered(head);
    return this;
}
```

### (30).AbstractChannelHandlerContext.invokeChannelRegistered
```
static void invokeChannelRegistered(final AbstractChannelHandlerContext next) {
    // next = DefaultChannelPipeline$HeadContext#0
    // executor = io.netty.channel.nio.NioEventLoop
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {// true
        // 
        next.invokeChannelRegistered();
    } else {
        executor.execute(new Runnable() {
            @Override
            public void run() {
                next.invokeChannelRegistered();
            }
        });
    }
} // end invokeChannelRegistered


private void invokeChannelRegistered() {
    if (invokeHandler()) { // true
        try {
            // 调用:channelRegistered(ChannelHandlerContext)方法
            ((ChannelInboundHandler) handler()).channelRegistered(this);
        } catch (Throwable t) {
            invokeExceptionCaught(t);
        }
    } else {
        fireChannelRegistered();
    }
} // end

```
### (31). 总结
    1. 创建NioEventLoopGroup,内部持有多个:NioEventLoop/Selector.
    2. 创建NioServerSocketChannel(内部持有:ChannelPipeline)
    3. 调用:NioEventLoop.register(NioServerSocketChannel)
    4. NioEventLoop委托给(NioServerSocketChannel)
        AbstractNioMessageChannel$NioMessageUnsafe.register
    5. AbstractNioMessageChannel$NioMessageUnsafe.register
        5.1 创建Runnable -> AbstractChannel$AbstractUnsafe.register0
        5.2 委托给:NioEventLoop.execute(Runnable)异步提交任务.
    6. NioEventLoop.execute
        6.1 添加Runnable到NioEventLoop.taskQueue中
        6.2 启动线程(SingleThreadEventExecutor.run())
    7. NioEventLoop.run
        7.1 对事件进行处理,但没有事件时,执行所有的任务	
            (SingleThreadEventExecutor.runAllTasks)
        7.2 SingleThreadEventExecutor.runAllTasks会从SingleThreadEventExecutor.taskQueue中拉取任务,并运行.
    8. SingleThreadEventExecutor.taskQueue任务有如下几个
        8.1 AbstractChannel$AbstractUnsafe.register0
        8.2 ServerBootstrap.init() 方法内部创建Runnable:Runnable(ServerBootstrapAcceptor)