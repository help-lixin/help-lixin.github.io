---
layout: post
title: 'Canal Server源码之七(CanalEventStore)'
date: 2019-10-01
author: 李新
tags: Canal
---

### (1). 概述

> 上一节,我分析到了:EntryEventSink,在它的内部最终会调用:CanalEventStore操作.

### (2). 准备工作
> 以一条INSERT语句进行跟踪  

```
INSERT INTO `test2`.`user` (`id`, `nick`, `phone`, `password`, `email`) VALUES ('6', '赵六', '8888888', '88888888', '123@126.com');
```

### (3). 程序入口
```
public class EntryEventSink 
       extends AbstractCanalEventSink<List<CanalEntry.Entry>> 
       implements CanalEventSink<List<CanalEntry.Entry>> {

    protected boolean doSink(List<Event> events) {

        // PrometheusCanalEventDownStreamHandler 
        // HeartBeatEntryEventHandler 

        // ... ...
        // 阻塞启动时间
        long blockingStart = 0L;
        // 记录调用:eventStore.tryPut不成功的次数
        int fullTimes = 0;
        do {
            // ****************************************************************
            // 5.尝试添加数据(MemoryEventStoreWithBuffer.tryPut)
            // ****************************************************************
            if (eventStore.tryPut(events)) { // 添加数据成功
                if (fullTimes > 0) {
                    eventsSinkBlockingTime.addAndGet(System.nanoTime() - blockingStart);
                }

                // 调用所有的:CanalEventDownStreamHandler的after方法
                for (CanalEventDownStreamHandler<List<Event>> handler : getHandlers()) {
                    events = handler.after(events);
                }
                // 退出do while循环
                return true;
            } else {  // 添加数据不成功,进行重试
                // blockingStart为系统时间
                if (fullTimes == 0) {
                    blockingStart = System.nanoTime();
                }

                // fullTimes进行递增
                // 如果:fullTimes小于等于3,则让线程yield.
                // 如果:fullTimes大于3,则让线程睡10秒针.
                applyWait(++fullTimes);

                // 当失败的调用了100次
                if (fullTimes % 100 == 0) { 
                    long nextStart = System.nanoTime();
                    eventsSinkBlockingTime.addAndGet(nextStart - blockingStart);
                    blockingStart = nextStart;
                }
            }

            // 调用所有的:CanalEventDownStreamHandler的retry方法
            for (CanalEventDownStreamHandler<List<Event>> handler : getHandlers()) {
                events = handler.retry(events);
            }

            // 添加数据不成功的情况下,好像就是执行了等待,然后,再次进入do while循环
        } while (running && !Thread.interrupted());
        // ... ...
    }
}// end doSink

private void applyWait(int fullTimes) {
    int newFullTimes = fullTimes > maxFullTimes ? maxFullTimes : fullTimes;
    if (fullTimes <= 3) { // 3次以内
        Thread.yield();
    } else { // 超过3次，最多只sleep 10ms
        LockSupport.parkNanos(1000 * 1000L * newFullTimes);
    }
}// end applyWait
```

### (4). 查看CanalEventStore的实现(XML配置)
```
<bean id="eventStore" class="com.alibaba.otter.canal.store.memory.MemoryEventStoreWithBuffer">
    <property name="bufferSize" value="${canal.instance.memory.buffer.size:16384}" />
    <property name="bufferMemUnit" value="${canal.instance.memory.buffer.memunit:1024}" />
    <property name="batchMode" value="${canal.instance.memory.batch.mode:MEMSIZE}" />
    <property name="ddlIsolation" value="${canal.instance.get.ddl.isolation:false}" />
    <property name="raw" value="${canal.instance.memory.rawEntry:true}" />
</bean>
```
### (5). MemoryEventStoreWithBuffer.tryPut
```
public boolean tryPut(List<Event> data) throws CanalStoreException {
    // 数据为空,直接返回
    if (data == null || data.isEmpty()) {
        return true;
    }

    // 加锁
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        // 检查是否有空位
        if (!checkFreeSlotAt(putSequence.get() + data.size())) {
            return false;
        } else {
            // ************************************************************
            // 6. MemoryEventStoreWithBuffer.doPut
            // ************************************************************
            doPut(data);
            return true;
        }
    } finally {
        lock.unlock();
    }
}
```
### (6). MemoryEventStoreWithBuffer.doPut
```
private void doPut(List<Event> data) {
    // 获取当前填充数据到了哪个索引
    long current = putSequence.get();
    // 当前的索引 + 要填充数据的数量
    long end = current + data.size();

    // 假设这是第一次请求:
    // putSequence = -1
    // data.size = 1
    // 即:current = -1  end=0

    // 先写数据，再更新对应的cursor,并发度高的情况，putSequence会被get请求可见，拿出了ringbuffer中的老的Entry值
    // 从上次的putSequence,开始填充数据.
    for (long next = current + 1; next <= end; next++) {
        // 填充数组对应的下标
        entries[getIndex(next)] = data.get((int) (next - current - 1));
    }


    // 告诉:putSequence已经写到了哪个位置
    putSequence.set(end);

    // 记录一下gets memsize信息，方便快速检索
    if (batchMode.isMemSize()) { // true
        long size = 0;
        // 计算这一批event所占的内存大小.
        for (Event event : data) {
            size += calculateSize(event);
        }
        // 存储这一批event所占用的内存大小.
        putMemSize.getAndAdd(size);
    }

    // 计算这一批event所影响的行数,并存储到:putTableRows,同时,设置:putExecTime为event里的最后一条的时间.
    profiling(data, OP.PUT);
    // tell other threads that store is not empty

    notEmpty.signal();
}// end doPut
```

### (7). UML
!["CanalEventStore类结构图"](/assets/canal/imgs/CanalEventStore-class.jpg)

!["CanalEventStore时序图"](/assets/canal/imgs/CanalEventStore-seq.jpg)


### (8). 总结
> 1. CanalEventStore内部为一个数组.实际上是学习了:Disruptor,通过下标存取数据.   
> 2. CanalEventStore接受EntryEventSink的请求.   
> 2. 将Event添加到数组中.   
