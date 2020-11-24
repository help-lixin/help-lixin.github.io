---
layout: post
title: 'Canal Server源码之五(MysqlEventParser-2)'
date: 2019-10-01
author: 李新
tags: Canal
---

### (1). 概述

> 上一节分析了MysqlEventParser的启动过程,把线程部份留出来在这里讲,最主要原因是UML到时有点长,不太利于阅读.

### (2). AbstractEventParser.start
> 上一节分析了在AbstractEventParser.start()方法内部创建了一个线程,并启动.现在用着重点分析线程内容.

```
parseThread = new Thread(new Runnable() {

    public void run() {
        MDC.put("destination", String.valueOf(destination));
        ErosaConnection erosaConnection = null;
        while (running) {
            try {
                // 开始执行replication
                // 构造Erosa连接
                // **********************3.MysqlEventParser.buildErosaConnection()*****************************
                erosaConnection = buildErosaConnection();


                // 启动一个心跳线程
                // **********************4.AbstractEventParser.startHeartBeat()*****************************
                startHeartBeat(erosaConnection);


                // 执行dump前的准备工作
                // **********************5.MysqlEventParser.preDump()*****************************
                preDump(erosaConnection);

                // 连接(此时,查看MySQL会发现,多出一个线程)
                erosaConnection.connect();// 链接

                long queryServerId = erosaConnection.queryServerId();
                if (queryServerId != 0) {
                    serverId = queryServerId;
                }
                

                // 获取最后的位置信息
                // **********************这一部份内容太多,留到下一节进行讲解*****************************
                long start = System.currentTimeMillis();
                logger.warn("---> begin to find start position, it will be long time for reset or first position");
                EntryPosition position = findStartPosition(erosaConnection);
            } catch (TableIdNotFoundException e) {
                exception = e;
                // 特殊处理TableIdNotFound异常,出现这样的异常，一种可能就是起始的position是一个事务当中，导致tablemap
                // Event时间没解析过
                needTransactionPosition.compareAndSet(false, true);
                logger.error(String.format("dump address %s has an error, retrying. caused by ",
                    runningInfo.getAddress().toString()), e);
            } catch (Throwable e) {
                processDumpError(e);
                exception = e;
                if (!running) {
                    if (!(e instanceof java.nio.channels.ClosedByInterruptException || e.getCause() instanceof java.nio.channels.ClosedByInterruptException)) {
                        throw new CanalParseException(String.format("dump address %s has an error, retrying. ",
                            runningInfo.getAddress().toString()), e);
                    }
                } else {
                    logger.error(String.format("dump address %s has an error, retrying. caused by ",
                        runningInfo.getAddress().toString()), e);
                    sendAlarm(destination, ExceptionUtils.getFullStackTrace(e));
                }
                if (parserExceptionHandler != null) {
                    parserExceptionHandler.handle(e);
                }
            } finally {
                // 重新置为中断状态
                Thread.interrupted();
                // 关闭一下链接
                afterDump(erosaConnection);
                try {
                    if (erosaConnection != null) {
                        erosaConnection.disconnect();
                    }
                } catch (IOException e1) {
                    if (!running) {
                        throw new CanalParseException(String.format("disconnect address %s has an error, retrying. ",
                            runningInfo.getAddress().toString()),
                            e1);
                    } else {
                        logger.error("disconnect address {} has an error, retrying., caused by ",
                            runningInfo.getAddress().toString(),
                            e1);
                    }
                }
            }

        } //end while

        MDC.remove("destination");
    }// end run
});
```
### (3). MysqlEventParser.buildErosaConnection()
```
protected ErosaConnection buildErosaConnection() {
    // 1.buildMysqlConnection
    return buildMysqlConnection(this.runningInfo);
} // end buildErosaConnection

// 2.buildMysqlConnection
private MysqlConnection buildMysqlConnection(AuthenticationInfo runningInfo) {
    MysqlConnection connection = new MysqlConnection(
        // runningInfo.getAddress() = "127.0.0.1:3306"
        runningInfo.getAddress(),
        // runningInfo.getUsername() = "canal"
        runningInfo.getUsername(),
        // runningInfo.getPassword() = "canal"
        runningInfo.getPassword(),
        // connectionCharsetNumber = 33
        connectionCharsetNumber,
        // runningInfo.getDefaultDatabaseName() = ""
        runningInfo.getDefaultDatabaseName());
    // 16384    
    connection.getConnector().setReceiveBufferSize(receiveBufferSize);
    // 16384
    connection.getConnector().setSendBufferSize(sendBufferSize);
    // 30 * 1000
    connection.getConnector().setSoTimeout(defaultConnectionTimeoutInSeconds * 1000);
    // "UTF-8"
    connection.setCharset(connectionCharset);
    // 0
    connection.setReceivedBinlogBytes(receivedBinlogBytes);

    // 随机生成slaveId
    if (this.slaveId <= 0) {
        this.slaveId = generateUniqueServerId();
    }
    connection.setSlaveId(this.slaveId);
    return connection;
} //end buildMysqlConnection

```
### (4). AbstractEventParser.startHeartBeat()
```
protected void startHeartBeat(ErosaConnection connection) {
    lastEntryTime = 0L; // 初始化
    // lazy初始化一下
    if (timer == null) { //true
        // destination = example , address = /127.0.0.1:3306 , HeartBeatTimeTask
        String name = String.format("destination = %s , address = %s , HeartBeatTimeTask",
            destination,
            runningInfo == null ? null : runningInfo.getAddress().toString());

        synchronized (AbstractEventParser.class) {
            // synchronized (MysqlEventParser.class) {
            // why use MysqlEventParser.class, u know, MysqlEventParser is
            // the child class 4 AbstractEventParser,
            // do this is ...
            if (timer == null) {
                // 创建定时器
                timer = new Timer(name, true);
            }
        }
    } //end 

    // fixed issue #56，避免重复创建heartbeat线程
    if (heartBeatTimerTask == null) { // true
        // ********************************************************
        // 调用:buildHeartBeatTimeTask构建出一个HeartBeatTimeTask
        heartBeatTimerTask = buildHeartBeatTimeTask(connection);

        // interval = 3
        Integer interval = detectingIntervalInSeconds;
        // 延迟3秒,每隔3秒调度一次:heartBeatTimerTask
        timer.schedule(heartBeatTimerTask, interval * 1000L, interval * 1000L);
        logger.info("start heart beat.... ");
    }
}// end startHeartBeat


protected TimerTask buildHeartBeatTimeTask(ErosaConnection connection) {
    return new TimerTask() {
        public void run() {
            try {
                if (exception == null || lastEntryTime > 0) {
                    // 如果未出现异常，或者有第一条正常数据
                    long now = System.currentTimeMillis();
                    long inteval = (now - lastEntryTime) / 1000;
                    if (inteval >= detectingIntervalInSeconds) {
                        Header.Builder headerBuilder = Header.newBuilder();
                        headerBuilder.setExecuteTime(now);
                        Entry.Builder entryBuilder = Entry.newBuilder();
                        entryBuilder.setHeader(headerBuilder.build());
                        entryBuilder.setEntryType(EntryType.HEARTBEAT);
                        Entry entry = entryBuilder.build();
                        // 提交到sink中，目前不会提交到store中，会在sink中进行忽略
                        consumeTheEventAndProfilingIfNecessary(Arrays.asList(entry));
                    }
                }

            } catch (Throwable e) {
                logger.warn("heartBeat run failed ", e);
            }
        }

    };
} //end buildHeartBeatTimeTask
```
### (5). MysqlEventParser.preDump() 
```
protected void preDump(ErosaConnection connection) {
    // 如果不是MySqlConnection则抛出异常
    if (!(connection instanceof MysqlConnection)) { // false
        throw new CanalParseException("Unsupported connection type : " + connection.getClass().getSimpleName());
    }

    
    if (binlogParser != null && 
        binlogParser instanceof LogEventConvert) { // true
        // 在现在的连接上,创建出一个新的连接
        metaConnection = (MysqlConnection) connection.fork();
        try {
            // 发起连接
            // 这时在MySQL(show processlist)里就能看到有一个线程与MySQL连接了,在这里先不讲解,如何连接的,后面会有一节,专门讲解.
            metaConnection.connect();
        } catch (IOException e) {
            throw new CanalParseException(e);
        }

        // 判断是否支持的binlog类型 
        // supportBinlogFormats = [ROW, STATEMENT, MIXED]
        if (supportBinlogFormats != null && supportBinlogFormats.length > 0) {
            // 调用MySQL,执行:show variables like 'binlog_format';
            // 获得MySQL设置的binlog_format=ROW
            BinlogFormat format = ((MysqlConnection) metaConnection).getBinlogFormat();
            boolean found = false;
            for (BinlogFormat supportFormat : supportBinlogFormats) {
                if (supportFormat != null && format == supportFormat) {
                    found = true;
                    break;
                }
            }

            if (!found) {
                throw new CanalParseException("Unsupported BinlogFormat " + format);
            }
        }

        // 判断是否支持的image
        // supportBinlogImages = [FULL, MINIMAL, NOBLOB]
        if (supportBinlogImages != null && supportBinlogImages.length > 0) {
            BinlogImage image = ((MysqlConnection) metaConnection).getBinlogImage();
            boolean found = false;
            for (BinlogImage supportImage : supportBinlogImages) {
                if (supportImage != null && image == supportImage) {
                    found = true;
                    break;
                }
            }

            if (!found) {
                throw new CanalParseException("Unsupported BinlogImage " + image);
            }
        }

        
        // tableMetaTSDB在setEnableTsdb(true)时创建的
        // com.alibaba.otter.canal.parse.inbound.mysql.tsdb.DatabaseTableMeta
        if (tableMetaTSDB != null && tableMetaTSDB instanceof DatabaseTableMeta) {
            // true

            // 对元数据管理对象进行赋值.
            ((DatabaseTableMeta) tableMetaTSDB).setConnection(metaConnection);
            ((DatabaseTableMeta) tableMetaTSDB).setFilter(eventFilter);
            ((DatabaseTableMeta) tableMetaTSDB).setBlackFilter(eventBlackFilter);
            ((DatabaseTableMeta) tableMetaTSDB).setSnapshotInterval(tsdbSnapshotInterval);
            ((DatabaseTableMeta) tableMetaTSDB).setSnapshotExpire(tsdbSnapshotExpire);
            // 初始化
            ((DatabaseTableMeta) tableMetaTSDB).init(destination);
        }

        // 创建:TableMetaCache包裹着:MysqlConnection和TableMetaTSDB
        // 获取元数据时,先从TableMetaTSDB中获取,获取不到再从:MysqlConnection中获取
        tableMetaCache = new TableMetaCache(metaConnection, tableMetaTSDB);

        // 给LogEventConvert设置表元数据缓存
        ((LogEventConvert) binlogParser).setTableMetaCache(tableMetaCache);
    }

}// end preDump
```
### (7). UML图

!["MysqlEventParser dump过程"](/assets/canal/imgs/MysqlEventParser-uml-sequence-thread.jpg)


### (8). 总结
> 1. 创建MysqlConnection   
> 2. 创建心跳定时任务    
> 3. 根据MysqlConnection fork出一个新的MysqlConnection,用于元数据管理
>    canal server配置文件(canal.properties)   
>       // 配置canal支持的binlog.format格式:      
>       canal.instance.binlog.format = ROW,STATEMENT,MIXED        
>       // 配置canal支持的binlog.image       
>       canal.instance.binlog.image = FULL,MINIMAL,NOBLOB        
>    MysqlConnection向MySQL发送SQL语句:<font color='red'>show variables like 'binlog_format';</font>,查看canal server是否支持. 
>    MysqlConnection向MySQL发送SQL语句:<font color='red'>show variables like 'binlog_row_image';</font>,查看canal server是否支持.  
> 4. 后面的内容太多了,留到下一节讲解
