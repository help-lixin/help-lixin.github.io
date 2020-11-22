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
                long start = System.currentTimeMillis();
                logger.warn("---> begin to find start position, it will be long time for reset or first position");
                // 执行dump前的准备工作
                // **********************6.MysqlEventParser.findStartPosition*****************************
                EntryPosition position = findStartPosition(erosaConnection);
                final EntryPosition startPosition = position;
                if (startPosition == null) {
                    throw new PositionNotFoundException("can't find start position for " + destination);
                }

                if (!processTableMeta(startPosition)) {
                    throw new CanalParseException("can't find init table meta for " + destination
                                                    + " with position : " + startPosition);
                }
                long end = System.currentTimeMillis();
                logger.warn("---> find start position successfully, {}", startPosition.toString() + " cost : "
                                                                            + (end - start)
                                                                            + "ms , the next step is binlog dump");
                // 重新链接，因为在找position过程中可能有状态，需要断开后重建
                erosaConnection.reconnect();

                final SinkFunction sinkHandler = new SinkFunction<EVENT>() {

                    private LogPosition lastPosition;

                    public boolean sink(EVENT event) {
                        try {
                            CanalEntry.Entry entry = parseAndProfilingIfNecessary(event, false);

                            if (!running) {
                                return false;
                            }

                            if (entry != null) {
                                exception = null; // 有正常数据流过，清空exception
                                transactionBuffer.add(entry);
                                // 记录一下对应的positions
                                this.lastPosition = buildLastPosition(entry);
                                // 记录一下最后一次有数据的时间
                                lastEntryTime = System.currentTimeMillis();
                            }
                            return running;
                        } catch (TableIdNotFoundException e) {
                            throw e;
                        } catch (Throwable e) {
                            if (e.getCause() instanceof TableIdNotFoundException) {
                                throw (TableIdNotFoundException) e.getCause();
                            }
                            // 记录一下，出错的位点信息
                            processSinkError(e,
                                this.lastPosition,
                                startPosition.getJournalName(),
                                startPosition.getPosition());
                            throw new CanalParseException(e); // 继续抛出异常，让上层统一感知
                        }
                    }

                };

                // 开始dump数据
                if (parallel) {
                    // build stage processor
                    multiStageCoprocessor = buildMultiStageCoprocessor();
                    if (isGTIDMode() && StringUtils.isNotEmpty(startPosition.getGtid())) {
                        // 判断所属instance是否启用GTID模式，是的话调用ErosaConnection中GTID对应方法dump数据
                        GTIDSet gtidSet = MysqlGTIDSet.parse(startPosition.getGtid());
                        ((MysqlMultiStageCoprocessor) multiStageCoprocessor).setGtidSet(gtidSet);
                        multiStageCoprocessor.start();
                        erosaConnection.dump(gtidSet, multiStageCoprocessor);
                    } else {
                        multiStageCoprocessor.start();
                        if (StringUtils.isEmpty(startPosition.getJournalName())
                            && startPosition.getTimestamp() != null) {
                            erosaConnection.dump(startPosition.getTimestamp(), multiStageCoprocessor);
                        } else {
                            erosaConnection.dump(startPosition.getJournalName(),
                                startPosition.getPosition(),
                                multiStageCoprocessor);
                        }
                    }
                } else {
                    if (isGTIDMode() && StringUtils.isNotEmpty(startPosition.getGtid())) {
                        // 判断所属instance是否启用GTID模式，是的话调用ErosaConnection中GTID对应方法dump数据
                        erosaConnection.dump(MysqlGTIDSet.parse(startPosition.getGtid()), sinkHandler);
                    } else {
                        if (StringUtils.isEmpty(startPosition.getJournalName())
                            && startPosition.getTimestamp() != null) {
                            erosaConnection.dump(startPosition.getTimestamp(), sinkHandler);
                        } else {
                            erosaConnection.dump(startPosition.getJournalName(),
                                startPosition.getPosition(),
                                sinkHandler);
                        }
                    }
                }
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

            // 出异常了，退出sink消费，释放一下状态
            eventSink.interrupt();
            transactionBuffer.reset();// 重置一下缓冲队列，重新记录数据
            binlogParser.reset();// 重新置位
            if (multiStageCoprocessor != null && multiStageCoprocessor.isStart()) {
                // 处理 RejectedExecutionException
                try {
                    multiStageCoprocessor.stop();
                } catch (Throwable t) {
                    logger.debug("multi processor rejected:", t);
                }
            }

            if (running) {
                // sleep一段时间再进行重试
                try {
                    Thread.sleep(10000 + RandomUtils.nextInt(10000));
                } catch (InterruptedException e) {
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

        // true    
        if (tableMetaTSDB != null && tableMetaTSDB instanceof DatabaseTableMeta) {
            // 对元数据管理对象进行赋值.
            ((DatabaseTableMeta) tableMetaTSDB).setConnection(metaConnection);
            ((DatabaseTableMeta) tableMetaTSDB).setFilter(eventFilter);
            ((DatabaseTableMeta) tableMetaTSDB).setBlackFilter(eventBlackFilter);
            ((DatabaseTableMeta) tableMetaTSDB).setSnapshotInterval(tsdbSnapshotInterval);
            ((DatabaseTableMeta) tableMetaTSDB).setSnapshotExpire(tsdbSnapshotExpire);
            // 初始化
            ((DatabaseTableMeta) tableMetaTSDB).init(destination);
        }

        // 创建元数据缓存
        tableMetaCache = new TableMetaCache(metaConnection, tableMetaTSDB);

        // 给LogEventConvert设置表元数据缓存
        ((LogEventConvert) binlogParser).setTableMetaCache(tableMetaCache);
    }

}// end preDump
```
### (6). MysqlEventParser.findStartPosition
```
// 1.findStartPosition
protected EntryPosition findStartPosition(ErosaConnection connection) throws IOException {
    if (isGTIDMode()) {  //false,暂时还未配置gtid
        // GTID模式下，CanalLogPositionManager里取最后的gtid，没有则取instanc配置中的
        LogPosition logPosition = getLogPositionManager().getLatestIndexBy(destination);
        if (logPosition != null) {
            // 如果以前是非GTID模式，后来调整为了GTID模式，那么为了保持兼容，需要判断gtid是否为空
            if (StringUtils.isNotEmpty(logPosition.getPostion().getGtid())) {
                return logPosition.getPostion();
            }
        }else {
            if (masterPosition != null && StringUtils.isNotEmpty(masterPosition.getGtid())) {
                return masterPosition;
            }
        }
    }

    //2.findStartPositionInternal
    EntryPosition startPosition = findStartPositionInternal(connection);
    if (needTransactionPosition.get()) {
        logger.warn("prepare to find last position : {}", startPosition.toString());
        Long preTransactionStartPosition = findTransactionBeginPosition(connection, startPosition);
        if (!preTransactionStartPosition.equals(startPosition.getPosition())) {
            logger.warn("find new start Transaction Position , old : {} , new : {}",
                startPosition.getPosition(),
                preTransactionStartPosition);
            startPosition.setPosition(preTransactionStartPosition);
        }
        needTransactionPosition.compareAndSet(true, false);
    }
    return startPosition;
} //end findStartPosition


// 3.findStartPositionInternal
protected EntryPosition findStartPositionInternal(ErosaConnection connection) {
    MysqlConnection mysqlConnection = (MysqlConnection) connection;

    // 1. 从本地内存中获取上一次binlogpoistion,获取不到,则,再从文件中获取.
    // 在这里典型的用到了:组合模式
    // logPositionManager = FailbackLogPositionManager
    LogPosition logPosition = logPositionManager.getLatestIndexBy(destination);

    // 2.找不到历史成功记录,代表第一次调用
    if (logPosition == null) { //true

        EntryPosition entryPosition = null;

        // <property name="masterPosition">
		//	<bean class="com.alibaba.otter.canal.protocol.position.EntryPosition">
		//		<property name="journalName" value="${canal.instance.master.journal.name}" />
		//		<property name="position" value="${canal.instance.master.position}" />
		//		<property name="timestamp" value="${canal.instance.master.timestamp}" />
		//		<property name="gtid" value="${canal.instance.master.gtid}" />
		//	</bean>
		// </property>
        // 针对master进行处理
        if (masterInfo != null && mysqlConnection.getConnector().getAddress().equals(masterInfo.getAddress())) { //true
            entryPosition = masterPosition;
        } else if (standbyInfo != null
                    && mysqlConnection.getConnector().getAddress().equals(standbyInfo.getAddress())) {  
            
            //针对standb进行处理
            entryPosition = standbyPosition;
        }

        if (entryPosition == null) { //false
            entryPosition = findEndPositionWithMasterIdAndTimestamp(mysqlConnection); // 默认从当前最后一个位置进行消费
        }

        // 判断一下是否需要按时间订阅
        if (StringUtils.isEmpty(entryPosition.getJournalName())) { //true
            // 如果没有指定binlogName，尝试按照timestamp进行查找
            
            // ****************************************************
            // 判断是否有配置时间点,如果有配置时间点,则按时间点进行查找
            // ****************************************************
            if (entryPosition.getTimestamp() != null && entryPosition.getTimestamp() > 0L) { 
                logger.warn("prepare to find start position {}:{}:{}",
                    new Object[] { "", "", entryPosition.getTimestamp() });
                return findByStartTimeStamp(mysqlConnection, entryPosition.getTimestamp());
            } else {

                
                logger.warn("prepare to find start position just show master status");

                // ****************************************************
                // 默认从当前最后一个位置进行消费
                // ****************************************************
                // 4.findEndPositionWithMasterIdAndTimestamp
                return findEndPositionWithMasterIdAndTimestamp(mysqlConnection); 
            }
        } else {
            if (entryPosition.getPosition() != null && entryPosition.getPosition() > 0L) {
                // 如果指定binlogName + offest，直接返回
                entryPosition = findPositionWithMasterIdAndTimestamp(mysqlConnection, entryPosition);
                logger.warn("prepare to find start position {}:{}:{}",
                    new Object[] { entryPosition.getJournalName(), entryPosition.getPosition(),
                            entryPosition.getTimestamp() });
                return entryPosition;
            } else {
                EntryPosition specificLogFilePosition = null;
                if (entryPosition.getTimestamp() != null && entryPosition.getTimestamp() > 0L) {
                    // 如果指定binlogName +
                    // timestamp，但没有指定对应的offest，尝试根据时间找一下offest
                    EntryPosition endPosition = findEndPosition(mysqlConnection);
                    if (endPosition != null) {
                        logger.warn("prepare to find start position {}:{}:{}",
                            new Object[] { entryPosition.getJournalName(), "", entryPosition.getTimestamp() });
                        specificLogFilePosition = findAsPerTimestampInSpecificLogFile(mysqlConnection,
                            entryPosition.getTimestamp(),
                            endPosition,
                            entryPosition.getJournalName(),
                            true);
                    }
                }

                if (specificLogFilePosition == null) {
                    // position不存在，从文件头开始
                    entryPosition.setPosition(BINLOG_START_OFFEST);
                    return entryPosition;
                } else {
                    return specificLogFilePosition;
                }
            }
        }
    } else {
        if (logPosition.getIdentity().getSourceAddress().equals(mysqlConnection.getConnector().getAddress())) {
            if (dumpErrorCountThreshold >= 0 && dumpErrorCount > dumpErrorCountThreshold) {
                // binlog定位位点失败,可能有两个原因:
                // 1. binlog位点被删除
                // 2.vip模式的mysql,发生了主备切换,判断一下serverId是否变化,针对这种模式可以发起一次基于时间戳查找合适的binlog位点
                boolean case2 = (standbyInfo == null || standbyInfo.getAddress() == null)
                                && logPosition.getPostion().getServerId() != null
                                && !logPosition.getPostion().getServerId().equals(findServerId(mysqlConnection));
                if (case2) {
                    long timestamp = logPosition.getPostion().getTimestamp();
                    long newStartTimestamp = timestamp - fallbackIntervalInSeconds * 1000;
                    logger.warn("prepare to find start position by last position {}:{}:{}", new Object[] { "", "",
                            logPosition.getPostion().getTimestamp() });
                    EntryPosition findPosition = findByStartTimeStamp(mysqlConnection, newStartTimestamp);
                    // 重新置为一下
                    dumpErrorCount = 0;
                    return findPosition;
                }

                Long timestamp = logPosition.getPostion().getTimestamp();
                if (isRdsOssMode() && (timestamp != null && timestamp > 0)) {
                    // 如果binlog位点不存在，并且属于timestamp不为空,可以返回null走到oss binlog处理
                    return null;
                }
            }
            // 其余情况
            logger.warn("prepare to find start position just last position\n {}",
                JsonUtils.marshalToString(logPosition));
            return logPosition.getPostion();
        } else {
            // 针对切换的情况，考虑回退时间
            long newStartTimestamp = logPosition.getPostion().getTimestamp() - fallbackIntervalInSeconds * 1000;
            logger.warn("prepare to find start position by switch {}:{}:{}", new Object[] { "", "",
                    logPosition.getPostion().getTimestamp() });
            return findByStartTimeStamp(mysqlConnection, newStartTimestamp);
        }
    }
}//end findStartPositionInternal

// 5.findEndPositionWithMasterIdAndTimestamp
protected EntryPosition findEndPositionWithMasterIdAndTimestamp(MysqlConnection connection) {
    MysqlConnection mysqlConnection = (MysqlConnection) connection;
    // 6. findEndPosition
    final EntryPosition endPosition = findEndPosition(mysqlConnection);
    if (tableMetaTSDB != null) {
        long startTimestamp = System.currentTimeMillis();
        // 8.
        return findAsPerTimestampInSpecificLogFile(mysqlConnection,
            startTimestamp,
            endPosition,
            endPosition.getJournalName(),
            true);
    } else {
        return endPosition;
    }
} //end findEndPositionWithMasterIdAndTimestamp

//7.findEndPosition
private EntryPosition findEndPosition(MysqlConnection mysqlConnection) {
    try {
        
        // 执行:MySql命令(show master status)

        // mysql> show master status\G;
        // ************** 1. row ***************
        //  File: mysql-bin.000019
        //  Position: 154
        //  Binlog_Do_DB: 
        //  Binlog_Ignore_DB: 
        //  Executed_Gtid_Set: 

        ResultSetPacket packet = mysqlConnection.query("show master status");
        List<String> fields = packet.getFieldValues();
        if (CollectionUtils.isEmpty(fields)) {
            throw new CanalParseException("command : 'show master status' has an error! pls check. you need (at least one of) the SUPER,REPLICATION CLIENT privilege(s) for this operation");
        }
        EntryPosition endPosition = new EntryPosition(fields.get(0), Long.valueOf(fields.get(1)));
        if (isGTIDMode() && fields.size() > 4) {
            endPosition.setGtid(fields.get(4));
        }
        return endPosition;
    } catch (IOException e) {
        throw new CanalParseException("command : 'show master status' has an error!", e);
    }
} //end findEndPosition

 private EntryPosition findAsPerTimestampInSpecificLogFile   
      (MysqlConnection mysqlConnection,
       // startTimestamp = 当前时间
       final Long startTimestamp,
       // 
       final EntryPosition endPosition,
       // searchBinlogFile = mysql-bin.000019
       final String searchBinlogFile,
       // true
       final Boolean justForPositionTimestamp) {
    final LogPosition logPosition = new LogPosition();
    try {
        // 销毁连接,并重新建立新的连接
        mysqlConnection.reconnect();

        // ******************************************************
        // 调用mysql,从指定的binlog开始获取
        // 开始遍历文件
        // ******************************************************
        mysqlConnection.seek(searchBinlogFile, 4L, endPosition.getGtid(), new SinkFunction<LogEvent>() {

            private LogPosition lastPosition;

            public boolean sink(LogEvent event) {
                EntryPosition entryPosition = null;
                try {
                    CanalEntry.Entry entry = parseAndProfilingIfNecessary(event, true);
                    if (justForPositionTimestamp && logPosition.getPostion() == null && event.getWhen() > 0) {
                        // 初始位点
                        entryPosition = new EntryPosition(searchBinlogFile,
                            event.getLogPos() - event.getEventLen(),
                            event.getWhen() * 1000,
                            event.getServerId());
                        entryPosition.setGtid(event.getHeader().getGtidSetStr());
                        logPosition.setPostion(entryPosition);
                    }

                    // 直接用event的位点来处理,解决一个binlog文件里没有任何事件导致死循环无法退出的问题
                    String logfilename = event.getHeader().getLogFileName();
                    // 记录的是binlog end offest,
                    // 因为与其对比的offest是show master status里的end offest
                    Long logfileoffset = event.getHeader().getLogPos();
                    Long logposTimestamp = event.getHeader().getWhen() * 1000;
                    Long serverId = event.getHeader().getServerId();

                    // 如果最小的一条记录都不满足条件，可直接退出
                    if (logposTimestamp >= startTimestamp) {
                        return false;
                    }

                    if (StringUtils.equals(endPosition.getJournalName(), logfilename)
                        && endPosition.getPosition() <= logfileoffset) {
                        return false;
                    }

                    if (entry == null) {
                        return true;
                    }

                    // 记录一下上一个事务结束的位置，即下一个事务的position
                    // position = current +
                    // data.length，代表该事务的下一条offest，避免多余的事务重复
                    if (CanalEntry.EntryType.TRANSACTIONEND.equals(entry.getEntryType())) {
                        entryPosition = new EntryPosition(logfilename, logfileoffset, logposTimestamp, serverId);
                        if (logger.isDebugEnabled()) {
                            logger.debug("set {} to be pending start position before finding another proper one...",
                                entryPosition);
                        }
                        logPosition.setPostion(entryPosition);
                        entryPosition.setGtid(entry.getHeader().getGtid());
                    } else if (CanalEntry.EntryType.TRANSACTIONBEGIN.equals(entry.getEntryType())) {
                        // 当前事务开始位点
                        entryPosition = new EntryPosition(logfilename, logfileoffset, logposTimestamp, serverId);
                        if (logger.isDebugEnabled()) {
                            logger.debug("set {} to be pending start position before finding another proper one...",
                                entryPosition);
                        }
                        entryPosition.setGtid(entry.getHeader().getGtid());
                        logPosition.setPostion(entryPosition);
                    }

                    lastPosition = buildLastPosition(entry);
                } catch (Throwable e) {
                    processSinkError(e, lastPosition, searchBinlogFile, 4L);
                }

                return running;
            }
        });

    } catch (IOException e) {
        logger.error("ERROR ## findAsPerTimestampInSpecificLogFile has an error", e);
    }

    if (logPosition.getPostion() != null) {
        return logPosition.getPostion();
    } else {
        return null;
    }
}// end findAsPerTimestampInSpecificLogFile

```
### (7). 

### (8). 

### (9). 

### (10). 
