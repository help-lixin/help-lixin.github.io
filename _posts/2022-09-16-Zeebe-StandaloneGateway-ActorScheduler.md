---
layout: post
title: 'Zeebe Gateway源码之StandaloneGateway创建ActorScheduler详解(二)' 
date: 2022-09-16
author: 李新
tags:  Zeebe
---

### (1). 概述
在前一小节,对StandaloneGateway创建AtomixCluster进行了源码剖析,还得继续往下看源码,这一小节,主要剖析的是与ActorScheduler相关的内容. 

### (2). StandaloneGateway.createActorScheduler
```
private ActorScheduler createActorScheduler(final GatewayCfg config) {
	return ActorScheduler.newActorScheduler()
	    // 1 设置CPU线程数
		.setCpuBoundActorThreadCount(config.getThreads().getManagementThreads())
		// 2. 设置IO线程数
		.setIoBoundActorThreadCount(0)
		// 3. 设置线程名称
		.setSchedulerName("gateway-scheduler")
		.setActorClock(clockConfig.getClock())
		// *************************************************************
		// 4. 构建:ActorScheduler
		// *************************************************************
		.build();
}
```
### (3). ActorScheduler.build
```
public ActorScheduler build() {
  // 初始化线程工厂
  initActorThreadFactory();
  // 初始化CPU线程组
  initCpuBoundActorThreadGroup();
  // 初始化IO线程数
  initIoBoundActorThreadGroup();
  // 初始化Executor
  initActorExecutor();
  return new ActorScheduler(this);
}
```
### (4). ActorScheduler.initActorThreadFactory
> ActorThreadFactory的主要职职责是创建:ActorThread线程.  

```
private void initActorThreadFactory() {
  if (actorThreadFactory == null) {
	  // 1. 创建DefaultActorThreadFactory
	actorThreadFactory = new DefaultActorThreadFactory();
  }
}  // end initActorThreadFactory


// 1.1 DefaultActorThreadFactory实现了:ActorThreadFactory,所以,需要先看下该接口有哪些签名
public interface ActorThreadFactory {
    ActorThread newThread(
        String name,
        int id,
        ActorThreadGroup threadGroup,
        TaskScheduler taskScheduler,
        ActorClock clock,
        ActorTimerQueue timerQueue);
} // end 

// 1.2 通过对ActorThreadFactory的分析DefaultActorThreadFactory的主要责任应该就是创建Thread,只是这个Thread是被Zeebe包装过的.
public static class DefaultActorThreadFactory implements ActorThreadFactory {
	@Override
	public ActorThread newThread(
		final String name,
		final int id,
		final ActorThreadGroup threadGroup,
		final TaskScheduler taskScheduler,
		final ActorClock clock,
		final ActorTimerQueue timerQueue) {
	  return new ActorThread(name, id, threadGroup, taskScheduler, clock, timerQueue);
	}
} 
```
### (5). ActorScheduler.initCpuBoundActorThreadGroup
```
// 1.1 初始化Cpu线程组
private void initCpuBoundActorThreadGroup() {
  if (cpuBoundActorGroup == null) {
	cpuBoundActorGroup = new CpuThreadGroup(this);
  }
} // end initCpuBoundActorThreadGroup



// **********************************************
// 1.2 CpuThreadGroup继承于:ActorThreadGroup,所以,要分析下这个抽象类的行为
// **********************************************
public abstract class ActorThreadGroup {
   // 线程组名称
  protected final String groupName;
  // 看类的名称应该是Thread的实现类,在这里我不详细剖析这个类了,就把它理解为Thread的增加吧,后面再进行详细剖析
  protected final ActorThread[] threads;
  // 不太理解这个类是干嘛的,先凉在这里.
  protected final MultiLevelWorkstealingGroup tasks;
  // 线程的数量
  protected final int numOfThreads;

  public ActorThreadGroup(
      final String groupName,
      final int numOfThreads,
      final int numOfQueuesPerThread,
      final ActorSchedulerBuilder builder) {
    
	// 设置线程名称
	this.groupName = groupName;
	// 设置线程数量
    this.numOfThreads = numOfThreads;
	// MultiLevelWorkstealingGroup内部结构实际上是参考了:Disruptor
	// 自己实现了一套环形队列,感觉Zeebe好像不太喜欢用开源已经成熟了的东西
	// 当然,也有一种可能性,那就是我没能理解他的精隋
    tasks = new MultiLevelWorkstealingGroup(numOfThreads, numOfQueuesPerThread);

    // 根据线程数量,创建N个线程数组,Zeebe并没有用Java提供的线程池,而是自己用数组创建了N个线程养着.
    threads = new ActorThread[numOfThreads];

    // **************************************************************************************
    // 对线程池里的线程进行初始化.
	// **************************************************************************************
    for (int t = 0; t < numOfThreads; t++) {
      final String threadName = String.format("%s-%d", groupName, t);
	  
	  // **************************************************************************************
	  // 调用子类(CpuThreadGroup)创建:TaskScheduler
	  // **************************************************************************************
      final TaskScheduler taskScheduler = createTaskScheduler(tasks, builder);

      final ActorThread thread =
          builder
              .getActorThreadFactory()
			  // 调用ActorThreadFactory创建线程.
              .newThread(
                  threadName,
                  t,
                  this,
                  taskScheduler,
                  builder.getActorClock(),
                  builder.getActorTimerQueue());

      threads[t] = thread;
    }
  }
    
  // 预留给子类自己去实现.
  protected TaskScheduler createTaskScheduler(final MultiLevelWorkstealingGroup tasks, final ActorSchedulerBuilder builder);
  
  // *****************************************************************
  // 从ActorThreadGroup类的方法签名上能看出来,这个类的主要目的在于提交task.
  // *****************************************************************  
  public void submit(final ActorTask actorTask) {
    final int level = getLevel(actorTask);

    final ActorThread current = ActorThread.current();
    if (current != null && current.getActorThreadGroup() == this) {
      tasks.submit(actorTask, level, current.getRunnerId());
    } else {
      final int threadId = ThreadLocalRandom.current().nextInt(numOfThreads);
      tasks.submit(actorTask, level, threadId);
      threads[threadId].hintWorkAvailable();
    }
  }
  

  // 调用所有线程的start方法
  public void start() {
    for (final ActorThread actorThread : threads) {
      actorThread.start();
    }
  }
}



// **********************************************
// CpuThreadGroup的职责如下:
// 1. 实现ActorThreadGroup.createTaskScheduler方法,一看名字:PriorityScheduler就知道这是一个有优先级的调度器.
// 2.  MultiLevelWorkstealingGroup内部实际是一个Queue来着的.
// 3. 设置调度的级别(HIGH:0 / REGULAR:1 / LOW:2 )
// **********************************************
public final class CpuThreadGroup extends ActorThreadGroup {

  public CpuThreadGroup(final ActorSchedulerBuilder builder) {
    super(
        String.format("%s-%s", builder.getSchedulerName(), "zb-actors"),
        builder.getCpuBoundActorThreadCount(),
        builder.getPriorityQuotas().length,
        builder);
  }

  @Override
  protected TaskScheduler createTaskScheduler(final MultiLevelWorkstealingGroup tasks, final ActorSchedulerBuilder builder) {
     // PriorityScheduler一看名字就知道这是一个有优先级的调度器,稍微看一下源码会发现:PriorityScheduler也是Zeebe自己封装的.
    return new PriorityScheduler(tasks::getNextTask, builder.getPriorityQuotas());
  } // end createTaskScheduler

  @Override
  protected int getLevel(final ActorTask actorTask) {
    return actorTask.getPriority();
  } // end getLevel
}
```
### (6). ActorScheduler.initActorExecutor
```
private void initActorExecutor() {
  if (actorExecutor == null) {
	actorExecutor = new ActorExecutor(this);
  }
} // end initActorExecutor


// **************************************************
// ActorExecutor相当于是Hold住两个线程组,是两个线程组的一个聚合类而已.
// 主要提供:启动线程池/提交任务/关闭线程池
// **************************************************
public final class ActorExecutor {
  private final ActorThreadGroup cpuBoundThreads;
  private final ActorThreadGroup ioBoundThreads;

  public ActorExecutor(final ActorSchedulerBuilder builder) {
    ioBoundThreads = builder.getIoBoundActorThreads();
    cpuBoundThreads = builder.getCpuBoundActorThreads();
  }

  public ActorFuture<Void> submitCpuBound(final ActorTask task) {
    return submitTask(task, cpuBoundThreads);
  }

  public ActorFuture<Void> submitIoBoundTask(final ActorTask task) {
    return submitTask(task, ioBoundThreads);
  }

  private ActorFuture<Void> submitTask(final ActorTask task, final ActorThreadGroup threadGroup) {
    if (task.getLifecyclePhase() != ActorLifecyclePhase.CLOSED) {
      throw new IllegalStateException("ActorTask was already submitted!");
    }
    final ActorFuture<Void> startingFuture = task.onTaskScheduled(this, threadGroup);

    threadGroup.submit(task);
    return startingFuture;
  }

  public void start() {
    cpuBoundThreads.start();
    ioBoundThreads.start();
  }

  public CompletableFuture<Void> closeAsync() {
    return CompletableFuture.allOf(ioBoundThreads.closeAsync(), cpuBoundThreads.closeAsync());
  }

  public ActorThreadGroup getCpuBoundThreads() {
    return cpuBoundThreads;
  }

  public ActorThreadGroup getIoBoundThreads() {
    return ioBoundThreads;
  }
} // end ActorExecutor

```
### (7). ActorScheduler构建器
```
public final class ActorScheduler implements AutoCloseable, ActorSchedulingService {
  private final AtomicReference<SchedulerState> state = new AtomicReference<>();
  private final ActorExecutor actorTaskExecutor;

  public ActorScheduler(final ActorSchedulerBuilder builder) {
    //  设置状态
    state.set(SchedulerState.NEW);
	// 包裹着两个ActorThreadGroup(CPU/IO)
    actorTaskExecutor = builder.getActorExecutor();
  } // end 
}  
```
### (8). 总结
ActorScheduler的内部维护着两个线程组(CPU密集型和IO密集型),这两个线程组是Zeebe自己开发的,并没有使用JDK自带的,因为,这两个线程组内部还维护着优先级来着的,更深的代码在这里不具体看了,毕竟,Zeebe后面还需要剖析源码的地方还很多,不能在这上面浪费太多时间了.   
