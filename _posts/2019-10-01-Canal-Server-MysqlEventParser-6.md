---
layout: post
title: 'Canal Server源码之五(MysqlEventParser-6)'
date: 2019-10-01
author: 李新
tags: Canal
---

### (1). 概述
!["MysqlMultiStageCoprocessor类结构图"](/assets/canal/imgs/MysqlMultiStageCoprocessor-class.jpg)

> 1. 建议,先参考:["Canal 如何使用Disruptor"](https://www.lixin.help/2019/10/07/Canal-Disruptor.html),在这里,我把Canal的业务逻辑给去除,纯粹的分析业务流转.       
> 2. 从类(MysqlMultiStageCoprocessor)结构图上看,MysqlMultiStageCoprocessor实际只有一个接口方法.   
> 3. 稍微说下:SinkStoreStage最终会该用类(EventTransactionBuffer.add)的方法,而这个类是在AbstractEventParser的构造器初始化的.   

### (2). MysqlConnection.dump
```
public void dump(
       String binlogfilename, 
       Long binlogPosition, 
       MultiStageCoprocessor coprocessor) throws IOException {
    
    // ... ...
    // ******************************************************************************
    // 3. 回调:MultiStageCoprocessor.publish
    //        MysqlMultiStageCoprocessor.publish
    // ******************************************************************************
    if (!coprocessor.publish(buffer)) {
        break;
    }
    // ... ...
}
```
### (3).  MysqlMultiStageCoprocessor.publish(LogBuffer)
> MysqlMultiStageCoprocessor.start()不再进行讲解,在上一节,我有做详细讲解.  

```
public boolean publish(LogBuffer buffer) {
    // 3.1 
    return this.publish(buffer, null);
} //end publish

/**
* 网络数据投递
*/
public boolean publish(LogEvent event) {
    return this.publish(null, event);
} //end publish


private boolean publish(LogBuffer buffer, LogEvent event) {

    // 判断是否有启动
    if (!isStart()) { 
        if (exception != null) {
            throw exception;
        }
        return false;
    }

    boolean interupted = false;
    long blockingStart = 0L;
    int fullTimes = 0;
    do {
        /**
            * 由于改为processor仅终止自身stage而不是stop，那么需要由incident标识coprocessor是否正常工作。
            * 让dump线程能够及时感知
            */
        if (exception != null) { // false
            throw exception;
        }
        try {
            // Disruptor发布事件的内容
            // 获取下一个序号
            long next = disruptorMsgBuffer.tryNext();
            // 获取序号对应的数据(为空对象)
            MessageEvent data = disruptorMsgBuffer.get(next);
            // 发布的事件可以是:LogBuffer/LogEvent  
            if (buffer != null) { // true
                data.setBuffer(buffer);
            } else {
                data.setEvent(event);
            }
            // 发布序号
            disruptorMsgBuffer.publish(next);

            // fullTimes = 0
            if (fullTimes > 0) {
                // eventsPublishBlockingTime = 0
                eventsPublishBlockingTime.addAndGet(System.nanoTime() - blockingStart);
            }
            // 退出do...while
            break;
        } catch (InsufficientCapacityException e) {
            if (fullTimes == 0) {
                blockingStart = System.nanoTime();
            }
            // park
            // LockSupport.parkNanos(1L);
            applyWait(++fullTimes);
            interupted = Thread.interrupted();
            if (fullTimes % 1000 == 0) {
                long nextStart = System.nanoTime();
                eventsPublishBlockingTime.addAndGet(nextStart - blockingStart);
                blockingStart = nextStart;
            }
        }
    } while (!interupted && isStart());

    // 返回线程的状态
    return isStart();
}// end publish

```
### (4). SimpleParserStage.onEvent
```
public void onEvent(
         MessageEvent event, 
         long sequence, 
         boolean endOfBatch) throws Exception {

    try {
        // 
        LogEvent logEvent = event.getEvent();
        // 如果logEvent为空,则代表发布的是:LogBuffer
        if (logEvent == null) { // true
            LogBuffer buffer = event.getBuffer();
            // 调用:com.taobao.tddl.dbsync.binlog.LogDecoder进行解码
            logEvent = decoder.decode(buffer, context);
            event.setEvent(logEvent);
        }

        // 从协头里获取事件类型
        int eventType = logEvent.getHeader().getType();
        TableMeta tableMeta = null;
        boolean needDmlParse = false;
        switch (eventType) {
            case LogEvent.WRITE_ROWS_EVENT_V1:
            case LogEvent.WRITE_ROWS_EVENT:
                tableMeta = logEventConvert.parseRowsEventForTableMeta((WriteRowsLogEvent) logEvent);
                needDmlParse = true;
                break;
            case LogEvent.UPDATE_ROWS_EVENT_V1:
            case LogEvent.PARTIAL_UPDATE_ROWS_EVENT:
            case LogEvent.UPDATE_ROWS_EVENT:
                tableMeta = logEventConvert.parseRowsEventForTableMeta((UpdateRowsLogEvent) logEvent);
                needDmlParse = true;
                break;
            case LogEvent.DELETE_ROWS_EVENT_V1:
            case LogEvent.DELETE_ROWS_EVENT:
                tableMeta = logEventConvert.parseRowsEventForTableMeta((DeleteRowsLogEvent) logEvent);
                needDmlParse = true;
                break;
            case LogEvent.ROWS_QUERY_LOG_EVENT:
                needDmlParse = true;
                break;
            default:
                // 调用: 
                //  com.alibaba.otter.canal.parse.inbound.mysql.dbsync.LogEventConvert
                // 解析LogEvent
                CanalEntry.Entry entry = logEventConvert.parse(event.getEvent(), false);
                event.setEntry(entry);
        }

        // 记录一下DML的表结构
        event.setNeedDmlParse(needDmlParse);
        event.setTable(tableMeta);
    } catch (Throwable e) {
        exception = new CanalParseException(e);
        throw exception;
    }
} //end onEvent

```
### (5). DmlParserStage.onEvent
```
public void onEvent(MessageEvent event) throws Exception {
    try {
        // 
        // 执行一条插入语句
        // INSERT INTO `test2`.`user` (`id`, `nick`, `phone`, `password`, `email`) VALUES ('4', '赵六', '8888888', '88888888', '123@126.com');
        // 对CanalEntry.Entry进行深度解析
        if (event.isNeedDmlParse()) {   // true
            // eventType = 30 
            int eventType = event.getEvent().getHeader().getType();
            CanalEntry.Entry entry = null;
            switch (eventType) {
                case LogEvent.ROWS_QUERY_LOG_EVENT:
                    entry = logEventConvert.parse(event.getEvent(), false);
                    break;
                default: // true
                    // 调用:com.alibaba.otter.canal.parse.inbound.mysql.dbsync.LogEventConvert 进行DML解析. 
                    // 单独解析dml事件
                    // ****************************************************
                    // 另再开一篇来讲解析单行事件吧
                    // ****************************************************
                    entry = logEventConvert.parseRowsEvent((RowsLogEvent) event.getEvent(), event.getTable());
            }

            event.setEntry(entry);
        }
    } catch (Throwable e) {
        exception = new CanalParseException(e);
        throw exception;
    }
} //end onEvent
```
### (6). SinkStoreStage.onEvent
```
public void onEvent(
    MessageEvent event, 
    long sequence, 
    boolean endOfBatch) throws Exception {

    try {
        // event.getEntry()不为空的情况下,回调:transactionBuffer进行flush.
        if (event.getEntry() != null) {
            // **************************************************
            // 这一部份的内容,另开一节来讲.
            // **************************************************
            transactionBuffer.add(event.getEntry());
        }


        // 针对MySQL半同步进行处理.向MySQL发送ack
        LogEvent logEvent = event.getEvent();
        if (connection instanceof MysqlConnection && logEvent.getSemival() == 1) {
            // semi ack回报
            ((MysqlConnection) connection).sendSemiAck(logEvent.getHeader().getLogFileName(),
                logEvent.getHeader().getLogPos());
        }

        // 清空Event里数据,防止内存泄露.
        // clear for gc
        event.setBuffer(null);
        event.setEvent(null);
        event.setTable(null);
        event.setEntry(null);
        event.setNeedDmlParse(false);
    } catch (Throwable e) {
        exception = new CanalParseException(e);
        throw exception;
    }
}// end onEvent
```
### (7). 总结
> 1. MysqlMultiStageCoprocessor.publish() 发布事件.   
> 2. SimpleParserStage.onEvent() 接受事件,进行简单的处理(DDL解析构造TableMeta、维护位点信息).  
> 3. DmlParserStage.onEvent()接受前面(SimpleParserStage)处理的事件,判断是否需要嵌套解析,如果需要则深度解析成:CanalEntry.Entry.    
> 4. SinkStoreStage.onEvent()接受前面(DmlParserStage)处理的事件,如果:CanalEntry.Entry不为空,则调用:EventTransactionBuffer.add()方法.EventTransactionBuffer类的信息后面再讲.可以透露一点:EventTransactionBuffer内部会调用:CanalEventSink.sink()方法.