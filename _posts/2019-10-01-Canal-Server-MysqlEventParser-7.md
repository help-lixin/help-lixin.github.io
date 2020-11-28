---
layout: post
title: 'Canal Server源码之五(MysqlEventParser-7)'
date: 2019-10-01
author: 李新
tags: Canal
---

### (1). 概述
> 前面的内容,跟踪到了:MysqlMultiStageCoprocessor$SinkStoreStage属于最后的消费者(Consumer),它会获取MessageEvent里的:CanalEntry.Entry添加到:EventTransactionBuffer中.   

### (2). MysqlMultiStageCoprocessor$SinkStoreStage.onEvent
```
public void onEvent(
    MessageEvent event, 
    long sequence, 
    boolean endOfBatch) throws Exception {
   
   // ... ...
   if (event.getEntry() != null) {
       // ****************************************************************
       // 4. 添加:CanalEntry.Entry
       // ****************************************************************
        transactionBuffer.add(event.getEntry());
    }
   // ... ...
}
```
### (3). EventTransactionBuffer类图
> EventTransactionBuffer的构造器要求必须指定一个回调函数:TransactionFlushCallback.我们要先看一下,EventTransactionBuffer在哪创建的(**AbstractEventParser的构造器里创建**).   

!["EventTransactionBuffer类结构图"](/assets/canal/imgs/EventTransactionBuffer-class.jpg)


```
public AbstractEventParser(){
    
    // 构建EventTransactionBuffer,同时指定回调函数:TransactionFlushCallback
    // 后面会回调该函数.
    transactionBuffer = new EventTransactionBuffer(new TransactionFlushCallback() {

        public void flush(List<CanalEntry.Entry> transaction) throws InterruptedException {

            // ****************************************************
            // 7. AbstractEventParser.consumeTheEventAndProfilingIfNecessary
            // ****************************************************

            boolean successed = consumeTheEventAndProfilingIfNecessary(transaction);
            if (!running) {
                return;
            }

            if (!successed) {
                throw new CanalParseException("consume failed!");
            }

            // 构建最后消费的position,并进行持久化.
            LogPosition position = buildLastTransactionPosition(transaction);
            if (position != null) { // 可能position为空
                logPositionManager.persistLogPosition(AbstractEventParser.this.destination, position);
            }
        }
    });
} //end 
```

### (4). AbstractEventParser.add
```
public void add(CanalEntry.Entry entry) throws InterruptedException {
    switch (entry.getEntryType()) {
        case TRANSACTIONBEGIN:   // 事务开始
            // *******************************************************************
            // 6. flush
            // *******************************************************************
            flush();// 刷新上一次的数据
            put(entry);
            break;
        case TRANSACTIONEND:     // 事务结束
            put(entry);
            // *******************************************************************
            // 6. flush
            // *******************************************************************
            flush();
            break;
        case ROWDATA:            // 增删改数据
            // *****************************************************************
            // 5. 添加到数组中:put
            // *****************************************************************
            put(entry);
            // 针对非DML的数据，直接输出，不进行buffer控制
            EventType eventType = entry.getHeader().getEventType();
            if (eventType != null && !isDml(eventType)) {
                flush();
            }
            break;
        case HEARTBEAT:           // 心跳
            // master过来的heartbeat，说明binlog已经读完了，是idle状态
            put(entry);
            flush();
            break;
        default:
            break;
    }//end switch
}// end add
```
### (5). EventTransactionBuffer.put
```
private void put(CanalEntry.Entry data) throws InterruptedException {
    // 首先检查数组是否有空位
    if (checkFreeSlotAt(putSequence.get() + 1)) {
        long current = putSequence.get();
        long next = current + 1;

        // 先写数据，再更新对应的cursor,并发度高的情况，putSequence会被get请求可见，拿出了ringbuffer中的老的Entry值
        entries[getIndex(next)] = data;
        putSequence.set(next);
    } else {
        flush();// buffer区满了，刷新一下
        put(data);// 继续加一下新数据
    }
}// end put
```
### (6). EventTransactionBuffer.flush
```
private void flush() throws InterruptedException {
    // 上一次flush的位置+1
    long start = this.flushSequence.get() + 1;
    // 上一次add的位置
    long end = this.putSequence.get();
    // flush位置应该要小于等于上次添加的位置来着的.

    if (start <= end) {
        // 拷贝start~end的数据,没有置空的过程,也就是说:Canal在学习Disruptor,玩数据覆盖.
        List<CanalEntry.Entry> transaction = new ArrayList<CanalEntry.Entry>();
        for (long next = start; next <= end; next++) {
            transaction.add(this.entries[getIndex(next)]);
        }

        // *******************************************************
        // 3. 调用创建EventTransactionBuffer时指定的:回调函数(TransactionFlushCallback)
        // *******************************************************
        flushCallback.flush(transaction);
        flushSequence.set(end);// flush成功后，更新flush位置
    }
} //end flush
```
### (7). AbstractEventParser.consumeTheEventAndProfilingIfNecessary
```
protected boolean consumeTheEventAndProfilingIfNecessary(
           List<CanalEntry.Entry> entrys) throws CanalSinkException,InterruptedException {
        long startTs = -1;

        // 查看是否开启了监控
        boolean enabled = getProfilingEnabled();
        if (enabled) {
            startTs = System.currentTimeMillis();
        }

        // *********************************************************
        // 调用:CanalEventSink.sink方法,这个方法里的内容另开一节来讲.
        // *********************************************************
        boolean result = eventSink.sink(
            entrys, 
            (runningInfo == null) ? null : runningInfo.getAddress(), 
            destination);

        if (enabled) {
            this.processingInterval = System.currentTimeMillis() - startTs;
        }

        // 消费者统计信息
        if (consumedEventCount.incrementAndGet() < 0) {
            consumedEventCount.set(0);
        }

        return result;
    }
```
### (8). UML时序图

 !["EventTransactionBuffer时序图"](/assets/canal/imgs/EventTransactionBuffer-seq.jpg)


### (9). 总结
> 1. MysqlMultiStageCoprocessor$SinkStoreStage.onEvent()接受事件.  
> 2. 获得事件(MessageEvent)里的内容(CanalEntry.Entry).   
> 3. 调用:EventTransactionBuffer.add(CanalEntry.Entry).    
> 4. EventTransactionBuffer.add方法的逻辑又会是什和呢?首先它会判断:CanalEntry.Entry是什么类型.
> 5. 如果:CanalEntry.Entry.getEntryType为:TRANSACTIONBEGIN/TRANSACTIONEND/HEARTBEAT就刷新缓存,并把CanalEntry.Entry添加到缓存中.这样意味着:每一次事务,刷新一次.   
> 6. 如果:CanalEntry.Entry.getEntryType为:ROWDATA,则CanalEntry.Entry添加到缓存中.  
> 7. 最终会:调用flush时,集合一批数据,并回调:TransactionFlushCallback.flush(List<CanalEntry.Entry>).  
> 8. TransactionFlushCallback.flush方法,会调用:<font color='red'>CanalEventSink.sink方法进行数据的发送.</font> 
