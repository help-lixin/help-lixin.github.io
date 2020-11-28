---
layout: post
title: 'Canal Server源码之六(CanalEventSink)'
date: 2019-10-01
author: 李新
tags: Canal
---

### (1). 概述
> 这一节,主要剖析:CanalEventSink,先看下CanalEventSink的类结构.

!["EntryEventSink 类结构图"](/assets/canal/imgs/EntryEventSink-class.jpg)

### (2). 程序入口(AbstractEventParser.consumeTheEventAndProfilingIfNecessary)
```
protected boolean consumeTheEventAndProfilingIfNecessary(
    List<CanalEntry.Entry> entrys) throws CanalSinkException,InterruptedException {

    // ... ...
    // ***********************************************************************
    // 3. 调用:EntryEventSink.sink对数据进行落地
    // ***********************************************************************
    eventSink.sink(
        entrys, 
        (runningInfo == null) ? null : runningInfo.getAddress(), 
        destination
    );
    // ... ...

}
```
### (3). EntryEventSink.sink
```
public boolean sink(
       List<CanalEntry.Entry> entrys, 
       InetSocketAddress remoteAddress, 
       String destination) throws CanalSinkException,InterruptedException {
        // ***********************************************************
        //  4. 委托给内部私有方法:sinkData
        // ***********************************************************
        return sinkData(entrys, remoteAddress);
}// end sink
```
### (4). EntryEventSink.sinkData
```
private boolean sinkData(
    List<CanalEntry.Entry> entrys, 
    InetSocketAddress remoteAddress) throws InterruptedException {

    boolean hasRowData = false;
    boolean hasHeartBeat = false;
    List<Event> events = new ArrayList<Event>();

    
    for (CanalEntry.Entry entry : entrys) {
        // 调用:CanalEventFilter.filter(),过滤用户配置要忽略的库或者表的相关事件.
        if (!doFilter(entry)) {  
            continue;
        }

        if (filterTransactionEntry
            && (entry.getEntryType() == EntryType.TRANSACTIONBEGIN || entry.getEntryType() == EntryType.TRANSACTIONEND)) {
            long currentTimestamp = entry.getHeader().getExecuteTime();
            // 基于一定的策略控制，放过空的事务头和尾，便于及时更新数据库位点，表明工作正常
            if (lastTransactionCount.incrementAndGet() <= emptyTransctionThresold
                && Math.abs(currentTimestamp - lastTransactionTimestamp) <= emptyTransactionInterval) {
                continue;
            } else {
                lastTransactionCount.set(0L);
                lastTransactionTimestamp = currentTimestamp;
            }
        }

        hasRowData |= (entry.getEntryType() == EntryType.ROWDATA);
        hasHeartBeat |= (entry.getEntryType() == EntryType.HEARTBEAT);
        Event event = new Event(new LogIdentity(remoteAddress, -1L), entry, raw);
        events.add(event);
    }

    // 存在row记录 或者 存在heartbeat记录，直接跳给后续处理
    if (hasRowData || hasHeartBeat) { // true
        // ********************************************************************
        // 5. EntryEventSink.doSink
        // ********************************************************************
        return doSink(events);
    } else {
        // 需要过滤的数据
        if (filterEmtryTransactionEntry && !CollectionUtils.isEmpty(events)) {
            long currentTimestamp = events.get(0).getExecuteTime();
            // 基于一定的策略控制，放过空的事务头和尾，便于及时更新数据库位点，表明工作正常
            if (Math.abs(currentTimestamp - lastEmptyTransactionTimestamp) > emptyTransactionInterval
                || lastEmptyTransactionCount.incrementAndGet() > emptyTransctionThresold) {
                lastEmptyTransactionCount.set(0L);
                lastEmptyTransactionTimestamp = currentTimestamp;
                return doSink(events);
            }
        }

        // 直接返回true，忽略空的事务头和尾
        return true;
    }// end else

} //endsinkData



protected boolean doFilter(CanalEntry.Entry entry) {
    // filter为抽象父类定义的:CanalEventFilter

    // 如果有定义:filter,并且:entry数据类型为:EntryType.ROWDATA
    if (filter != null && entry.getEntryType() == EntryType.ROWDATA) {
        // 从entry中获得schema和tablename
        String name = getSchemaNameAndTableName(entry);
        // 判断是否需要过滤,如果返回:true,则忽略这条消息(entry)
        boolean need = filter.filter(name);
        if (!need) {
            logger.debug("filter name[{}] entry : {}:{}",
                name,
                entry.getHeader().getLogfileName(),
                entry.getHeader().getLogfileOffset());
        }

        return need;
    } else {
        return true;
    }
} //end doFilter

```
### (5). EntryEventSink.doSink
```
protected boolean doSink(List<Event> events) {
    // CanalEventDownStreamHandler主要实现对数据的一些钩子函数,在数据落地之后调用:
    // 调用顺序如下:before->after->retry
    for (CanalEventDownStreamHandler<List<Event>> handler : getHandlers()) {
        events = handler.before(events);
    }

    long blockingStart = 0L;
    int fullTimes = 0;
    do {
        // ... ... 
        for (CanalEventDownStreamHandler<List<Event>> handler : getHandlers()) {
            events = handler.after(events);
        }

        // ... ....    
        for (CanalEventDownStreamHandler<List<Event>> handler : getHandlers()) {
            events = handler.retry(events);
        }

    } while (running && !Thread.interrupted());

    return false;
} //end doSink
```
### (6). UML时序图

!["EnteryEventSink执行时序图"](/assets/canal/imgs/EntryEventSink-seq.jpg)

### (7). 总结
> 1. 接受MysqlEventParser发送过来的数据组合(List<CanalEntry.Entry>).    
> 2. <font color='red'>遍历数据(List<CanalEntry.Entry>).判断是否符合用户定义(CanalEventFilter.filter)要排除的库或表,如果符合,直接抛弃该数据.如果不符合,将数据转换成:Event
> 3. 遍功上一步转换的:List<Event>.</font>   
> 4. <font color='red'>调用:CanalEventStore.store之前,先回调相应的钩子函数(CanalEventDownStreamHandler)</font>   
> 5. <font color='red'>钩子函数调用过程(before->after->retry).</font>   
> 6. EntryEventSink的职责基本完成,CanalEventStore的内容,留到另一节再剖析.