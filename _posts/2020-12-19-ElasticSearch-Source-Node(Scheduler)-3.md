---
layout: post
title: 'ElasticSearch 源码Node(Scheduler)(四)'
date: 2020-12-20
author: 李新
tags: ElasticSearch源码
---

### (1). 概述
> 这一节主要剖析:ES的线程池. 

### (2). 查看Scheduler的实现
!["Scheduler"](/assets/elasticsearch/imgs/es-scheduler.jpg)

### (3). 查看ExecutorBuilder类关系图
> ExecutorBuilder是构建线程池的策略,不同的策略,构建出不同的线程池.     
> AutoQueueAdjustingExecutorBuilder : 带有队列功能的线程池.       
> ScalingExecutorBuilder :    具有弹性伸缩的线程池.     
> FixedExecutorBuilder   :    固定大小的线程池.     

!["ExecutorBuilder"](/assets/elasticsearch/imgs/es-ExecutorBuilder-class.jpg)

### (4). 查看Scheduler的接口行为
```
public interface Scheduler {

    //  TODO ... ... 
    // ****************************************************************************
    // 定时调度,留给实现类去做实现
    // ****************************************************************************
    ScheduledCancellable schedule(Runnable command, TimeValue delay, String executor);
    

    default Cancellable scheduleWithFixedDelay(
        Runnable command, 
        TimeValue interval, 
        String executor) {
        
        return new ReschedulingRunnable(
                command, interval, executor, this, (e) -> {}, (e) -> {}
               );
    }// end scheduleWithFixedDelay

    interface Cancellable {

        boolean cancel();

        boolean isCancelled();

    } // end Cancellable


    interface ScheduledCancellable extends Delayed, Cancellable {}



    //  TODO ... ... 
}
```
### (5). ThreadPool

```
public ThreadPool(final Settings settings, final ExecutorBuilder<?>... customBuilders) {
    assert Node.NODE_NAME_SETTING.exists(settings);

    final Map<String, ExecutorBuilder> builders = new HashMap<>();

    // 根据CPU的数量来获取的 
    // 4
    final int availableProcessors = EsExecutors.numberOfProcessors(settings);
    // 2
    final int halfProcMaxAt5 = halfNumberOfProcessorsMaxFive(availableProcessors);
    // 2
    final int halfProcMaxAt10 = halfNumberOfProcessorsMaxTen(availableProcessors);
    // 128 
    final int genericThreadPoolMax = boundedBy(4 * availableProcessors, 128, 512);

    // 构建线程池的数量和名称.
    builders.put(Names.GENERIC, new ScalingExecutorBuilder(Names.GENERIC, 4, genericThreadPoolMax, TimeValue.timeValueSeconds(30)));
    builders.put(Names.WRITE, new FixedExecutorBuilder(settings, Names.WRITE, availableProcessors, 200));
    builders.put(Names.GET, new FixedExecutorBuilder(settings, Names.GET, availableProcessors, 1000));
    builders.put(Names.ANALYZE, new FixedExecutorBuilder(settings, Names.ANALYZE, 1, 16));
    builders.put(Names.SEARCH, new AutoQueueAdjustingExecutorBuilder(settings,
                    Names.SEARCH, searchThreadPoolSize(availableProcessors), 1000, 1000, 1000, 2000));
    builders.put(Names.SEARCH_THROTTLED, new AutoQueueAdjustingExecutorBuilder(settings,
        Names.SEARCH_THROTTLED, 1, 100, 100, 100, 200));
    builders.put(Names.MANAGEMENT, new ScalingExecutorBuilder(Names.MANAGEMENT, 1, 5, TimeValue.timeValueMinutes(5)));
    // no queue as this means clients will need to handle rejections on listener queue even if the operation succeeded
    // the assumption here is that the listeners should be very lightweight on the listeners side
    builders.put(Names.LISTENER, new FixedExecutorBuilder(settings, Names.LISTENER, halfProcMaxAt10, -1));
    builders.put(Names.FLUSH, new ScalingExecutorBuilder(Names.FLUSH, 1, halfProcMaxAt5, TimeValue.timeValueMinutes(5)));
    builders.put(Names.REFRESH, new ScalingExecutorBuilder(Names.REFRESH, 1, halfProcMaxAt10, TimeValue.timeValueMinutes(5)));
    builders.put(Names.WARMER, new ScalingExecutorBuilder(Names.WARMER, 1, halfProcMaxAt5, TimeValue.timeValueMinutes(5)));
    builders.put(Names.SNAPSHOT, new ScalingExecutorBuilder(Names.SNAPSHOT, 1, halfProcMaxAt5, TimeValue.timeValueMinutes(5)));
    builders.put(Names.FETCH_SHARD_STARTED,
            new ScalingExecutorBuilder(Names.FETCH_SHARD_STARTED, 1, 2 * availableProcessors, TimeValue.timeValueMinutes(5)));
    builders.put(Names.FORCE_MERGE, new FixedExecutorBuilder(settings, Names.FORCE_MERGE, 1, -1));
    builders.put(Names.FETCH_SHARD_STORE,
            new ScalingExecutorBuilder(Names.FETCH_SHARD_STORE, 1, 2 * availableProcessors, TimeValue.timeValueMinutes(5)));

    // 添加自定义的线程池信息
    for (final ExecutorBuilder<?> builder : customBuilders) {
        if (builders.containsKey(builder.name())) {
            throw new IllegalArgumentException("builder with name [" + builder.name() + "] already exists");
        }
        builders.put(builder.name(), builder);
    }

    // 不可变Map
    // builders.length = 22
    this.builders = Collections.unmodifiableMap(builders);

    // 创建线程上下文,包裹着配置文件
    threadContext = new ThreadContext(settings);

    // ******************************************************************
    // 前面创建了很多的线程池的配置信息(最小核心数,最大核心数,队列数...)
    // 这一步:
    // 1. 会根据上面的:ExecutorBuilder对象,创建出对应的:ExecutorService
    // 2. 创建出ExecutorHolder,包裹着:上一步的实例:ExecutorService
    // 3. 注意:ExecutorService(线程池可还没有启动)
    // ******************************************************************
    final Map<String, ExecutorHolder> executors = new HashMap<>();
    // 遍历所有的:ExecutorBuilder
    for (final Map.Entry<String, ExecutorBuilder> entry : builders.entrySet()) {
        // 获得配置项(最小核心数,最大核心数,队列数...)
        final ExecutorBuilder.ExecutorSettings executorSettings = 
                    entry.getValue().getSettings(settings);

        // *********************************************************
        // 在这里,我以:ScalingExecutorBuilder为例
        // ScalingExecutorBuilder.build
        // *********************************************************
        // 调用:build构建:ExecutorHolder,这个类,内部实际是Holder住:ExecutorService(JDK线程池)
        final ExecutorHolder executorHolder = 
                    entry.getValue().build(executorSettings, threadContext);
        if (executors.containsKey(executorHolder.info.getName())) {
            throw new IllegalStateException("duplicate executors with name [" + executorHolder.info.getName() + "] registered");
        }
        logger.debug("created thread pool: {}", entry.getValue().formatInfo(executorHolder.info));
        executors.put(entry.getKey(), executorHolder);
    }

    // 针对:same独特处理
    executors.put(Names.SAME, new ExecutorHolder(DIRECT_EXECUTOR, new Info(Names.SAME, ThreadPoolType.DIRECT)));
    // executors.size = 23
    
    // ThreadPool持有这个:executors
    this.executors = unmodifiableMap(executors);

    // 过滤出名称不是:same的Info信息
    // infos.size = 22
    final List<Info> infos =
            executors
                    .values()
                    .stream()
                    .filter(holder -> holder.info.getName().equals("same") == false)
                    .map(holder -> holder.info)
                    .collect(Collectors.toList());

    // ThreadPoolInfo包裹所有的:infos
    this.threadPoolInfo = new ThreadPoolInfo(infos);

    // *************************************************************
    // 8. 初始化定时调度(Scheduler.initScheduler)
    // *************************************************************
    this.scheduler = Scheduler.initScheduler(settings);

    // 创建:cached线程.
    TimeValue estimatedTimeInterval = ESTIMATED_TIME_INTERVAL_SETTING.get(settings);
    this.cachedTimeThread = new CachedTimeThread(EsExecutors.threadName(settings, "[timer]"), estimatedTimeInterval.millis());
    this.cachedTimeThread.start();
} // end ThreadPool


// 根据name获得相应的:ExecutorService
public ExecutorService executor(String name) {
    final ExecutorHolder holder = executors.get(name);
    if (holder == null) {
        throw new IllegalArgumentException("no executor service found for [" + name + "]");
    }
    return holder.executor();
}// end executor

// 调度
public ScheduledCancellable schedule(Runnable command, TimeValue delay, String executor) {
    // 如果executor不是:same
    if (!Names.SAME.equals(executor)) { 
        // 典型组合模式.
        // 通过:ThreadedRunnable包裹着:Runnable和Executor
        command = new ThreadedRunnable(command, executor(executor));
    }

    // scheduler.schedule(...)方法会返回:ScheduledFuture
    // ScheduledCancellableAdapter包裹着:ScheduledFuture,可实现取消功能.
    return new ScheduledCancellableAdapter(scheduler.schedule(command, delay.millis(), TimeUnit.MILLISECONDS));
}// end schedule

```
### (6).  ScalingExecutorBuilder.build
> ScalingExecutorBuilder是可弹性伸缩的线程池配置.   

```
ThreadPool.ExecutorHolder build(
        // 创建线程池需要的配置项
        final ScalingExecutorSettings settings, 
        // ThreadContext持有着的是配置文件(Settings).
        final ThreadContext threadContext) {

    // 存活时间
    TimeValue keepAlive = settings.keepAlive;
    // 核心线程数
    int core = settings.core;
    // 最大线程数
    int max = settings.max;

    // 构建:ThreadPool.Info,线程池的基本信息.
    final ThreadPool.Info info = new ThreadPool.Info(
                name(), 
                ThreadPool.ThreadPoolType.SCALING, 
                core, 
                max, 
                keepAlive, 
                null
            );// end ThreadPool.Info


    // 创建线程工厂,后台线程工厂.
    final ThreadFactory threadFactory = EsExecutors.daemonThreadFactory(
                    EsExecutors.threadName(settings.nodeName, name())
          );


    // **********************************************
    // 7. 调用:EsExecutors对象去创建线程池
    // **********************************************
    final ExecutorService executor =
        EsExecutors.newScaling(
                settings.nodeName + "/" + name(),
                core,
                max,
                keepAlive.millis(),
                TimeUnit.MILLISECONDS,
                threadFactory,
                threadContext);
    // 创建: ThreadPool.ExecutorHolder包裹着: ExecutorService 和    ThreadPool.Info     
    return new ThreadPool.ExecutorHolder(executor, info);
} // end build
```
### (7). EsExecutors.newScaling

```
public class EsExecutors {

    // 创建弹性线程池
    public static EsThreadPoolExecutor newScaling(String name, int min, int max, long keepAliveTime, TimeUnit unit,
                                                  ThreadFactory threadFactory, ThreadContext contextHolder) {
        // 创建弹性队列.
        ExecutorScalingQueue<Runnable> queue = new ExecutorScalingQueue<>();
        EsThreadPoolExecutor executor =
            // ForceQueuePolicy 为拒绝策略.
            new EsThreadPoolExecutor(name, min, max, keepAliveTime, unit, queue, threadFactory, new ForceQueuePolicy(), contextHolder);
        queue.executor = executor;
        return executor;
    }// end newScaling
}
```
### (8). Scheduler.initScheduler
```
static ScheduledThreadPoolExecutor initScheduler(Settings settings) {
    
    // 创建定时任务执行器
    final ScheduledThreadPoolExecutor scheduler = 
            new SafeScheduledThreadPoolExecutor(1,
                                                EsExecutors.daemonThreadFactory(settings, "scheduler",
                                                new EsAbortPolicy());
    scheduler.setExecuteExistingDelayedTasksAfterShutdownPolicy(false);
    scheduler.setContinueExistingPeriodicTasksAfterShutdownPolicy(false);
    scheduler.setRemoveOnCancelPolicy(true);
    return scheduler;
} // end initScheduler


class SafeScheduledThreadPoolExecutor extends ScheduledThreadPoolExecutor {

    public SafeScheduledThreadPoolExecutor(
            int corePoolSize, 
            ThreadFactory threadFactory, 
            RejectedExecutionHandler handler) {
        super(corePoolSize, threadFactory, handler);
    }
}
```
### (9).  总结
> 1. 为不同的名称:ThreadPool.Names,创建相应的线程池参数(ExecutorBuilder).临时存储在Map里.        
> 2. 遍历第一步的Map,调用:ExecutorBuilder.build方法创建:ExecutorHolder(ExecutorService).key=ThreadPool.Names,value=ExecutorHolder.    
> 3. 初始化ES定义的:Scheduler.initScheduler(xxx).     
> 4. Scheduler.schedule()相当于是代理所有的:ExecutorService去执行任务.   