---
layout: post
title: 'Canal Server源码之五(MysqlEventParser-3)'
date: 2019-10-01
author: 李新
tags: Canal
---

### (1). 概述

>  上一节,对MysqlEventParser内部的线程进行了分析,这一节,主要分析:查找启始position.

### (2). AbstractEventParser.run
```
parseThread = new Thread(new Runnable() {
    public void run() {
        ErosaConnection erosaConnection = null;
        while (running) {
            try {
                // 构造Erosa连接
                erosaConnection = buildErosaConnection();        

                // ....

                // 获取最后的位置信息
                long start = System.currentTimeMillis();
                logger.warn("---> begin to find start position, it will be long time for reset or first position");
                // *************************************************************
                // 2.从ErosaConnection中获得postion
                //   委托给子类:MysqlEventParser
                // *************************************************************
                EntryPosition position = findStartPosition(erosaConnection);
                final EntryPosition startPosition = position;
                if (startPosition == null) {
                    throw new PositionNotFoundException("can't find start position for " + destination);
                }

                if (!processTableMeta(startPosition)) {
                    throw new CanalParseException("can't find init table meta for " + destination + " with position : " + startPosition);
                }
                long end = System.currentTimeMillis();
                
                logger.warn("---> find start position successfully, {}", startPosition.toString() + " cost : "+ (end - start)+ "ms , the next step is binlog dump");
                
                // ...
            }catch(...){
               // ...
            }
        }
    }//end run 
}
```
### (3). MysqlEventParser.findStartPosition
```
protected EntryPosition findStartPosition(ErosaConnection connection) 
                                                     throws IOException {
    if (isGTIDMode()) { // false
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

    // **********************************************************************
    // 4.委托给:MysqlEventParser.findStartPositionInternal进行获取
    // **********************************************************************
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
}// end findStartPosition
```

### (4). MysqlEventParser.findStartPositionInternal

```
protected EntryPosition findStartPositionInternal(ErosaConnection connection) {
    MysqlConnection mysqlConnection = (MysqlConnection) connection;

    // 4.1 logPositionManager为日志位置记录管理器
    // com.alibaba.otter.canal.parse.index.FailbackLogPositionManager
    // FailbackLogPositionManager会先刷内存,后再刷持久设备
    LogPosition logPosition = logPositionManager.getLatestIndexBy(destination);

    // 找不到历史成功记录
    if (logPosition == null) {   // true
        EntryPosition entryPosition = null;

        // masterInfo = com.alibaba.otter.canal.parse.support.AuthenticationInfo@48da5682[address=/127.0.0.1:3306,username=canal,password=canal,defaultDatabaseName=,pwdPublicKey=<null>,enableDruid=false]
        // masterInfo为example/instance.properties中配置的master mysql信息

        // 以下内容实则是判断:mysqlConnection属于:master还是属于:standby
        if (masterInfo != null && 
            mysqlConnection.getConnector().getAddress()
            .equals(masterInfo.getAddress())) { 
            //     
            entryPosition = masterPosition;
        } else if (standbyInfo != null
                    && mysqlConnection.getConnector().getAddress().equals(standbyInfo.getAddress())) {
            entryPosition = standbyPosition;
        }

        
        if (entryPosition == null) { // false
            entryPosition = findEndPositionWithMasterIdAndTimestamp(mysqlConnection); // 默认从当前最后一个位置进行消费
        }

        
        // canal.instance.master.journal.name = entryPosition.getJournalName()
        // canal.instance.master.journal.name是在配置文件中指定的:binlogfileame
        // 判断配置文件是否有配置指定的binlogfilename
        // 如果没有指定binlogName，尝试按照timestamp进行查找
        if (StringUtils.isEmpty(entryPosition.getJournalName())) { // true

            // 如果没有指定binlogName，尝试按照timestamp进行查找
            // 判断下  
            // canal.instance.master.timestamp
            // 是否有配置这个值
            if (entryPosition.getTimestamp() != null && entryPosition.getTimestamp() > 0L) { //false
                logger.warn("prepare to find start position {}:{}:{}",
                    new Object[] { "", "", entryPosition.getTimestamp() });
                return findByStartTimeStamp(mysqlConnection, entryPosition.getTimestamp());
            } else {  // true
                logger.warn("prepare to find start position just show master status");
                
                // *************************************************************
                // 5.canal.instance.master.journal.name和canal.instance.master.timestamp配置都未指定,则调用:findEndPositionWithMasterIdAndTimestamp(mysqlConnection),从mysql最新位置进行订阅.
                // 默认从当前最后一个位置进行消费
                // *************************************************************
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
} //end findStartPositionInternal
```
### (5). MysqlEventParser.findEndPositionWithMasterIdAndTimestamp
```
protected EntryPosition findEndPositionWithMasterIdAndTimestamp(MysqlConnection connection) {
    MysqlConnection mysqlConnection = (MysqlConnection) connection;
    // *************************************************************
    // 6. 向MySQL服务器发送SQL(show master status),查找position信息
    // *************************************************************
    final EntryPosition endPosition = findEndPosition(mysqlConnection);

    if (tableMetaTSDB != null) { // true
        long startTimestamp = System.currentTimeMillis();
        // *************************************************************
        // 7. 在给定的binlog文件(mysql-bin.000021)中,搜索指定的时间.
        // *************************************************************
        return findAsPerTimestampInSpecificLogFile(mysqlConnection,
            startTimestamp,
            endPosition,
            endPosition.getJournalName(),
            true);
    } else {
        return endPosition;
    }
}// end findEndPositionWithMasterIdAndTimestamp
```
### (6). MysqlEventParser.findEndPosition
```
private EntryPosition findEndPosition(MysqlConnection mysqlConnection) {
    try {
        // 6.1 向MySQL发送SQL语句(show master status)
        ResultSetPacket packet = mysqlConnection.query("show master status");

        // fields = [mysql-bin.000021, 154, , , ]
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
}// end findEndPosition
```
### (7). MysqlEventParser.findAsPerTimestampInSpecificLogFile
```
private EntryPosition findAsPerTimestampInSpecificLogFile(
        MysqlConnection mysqlConnection,
        // 1606202067833
        final Long startTimestamp,
        // EntryPosition[included=false,journalName=mysql-bin.000021,position=154,serverId=<null>,gtid=<null>,timestamp=<null>]
        final EntryPosition endPosition,
        // mysql-bin.000021
        final String searchBinlogFile,
        // true
        final Boolean justForPositionTimestamp) {

    final LogPosition logPosition = new LogPosition();
    try {
        mysqlConnection.reconnect();
        // ************************************************************
        // 8. 开始遍历文件
        // ************************************************************
        mysqlConnection.seek(
                             searchBinlogFile, 
                             4L, 
                             endPosition.getGtid(), 
                             // 回调函数
                             new SinkFunction<LogEvent>() {

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
} //end findAsPerTimestampInSpecificLogFile
```
### (8). MysqlConnection.seek
```
public void seek(String binlogfilename, Long binlogPosition, String gtid, SinkFunction func) throws IOException {

    // updateSettings更新MySqlConnection相关信息
    // set wait_timeout=9999999;
    // set net_write_timeout=1800;
    // set net_read_timeout=1800;
    // set names 'binary';
    // set @master_binlog_checksum= @@global.binlog_checksum;
    // set @slave_uuid=uuid();
    // SET @mariadb_slave_capability='4';
    // SET @master_heartbeat_period=15000000000;
    updateSettings();

    // 获取binlog_checksum
    // select @@global.binlog_checksum; 
    loadBinlogChecksum();

    // 向MySQL发起:binlog dump请求
    sendBinlogDump(binlogfilename, binlogPosition);

    // ***************************************************************
    // 日志抓取功能,留到下一节再讲.
    // ***************************************************************
    DirectLogFetcher fetcher = new DirectLogFetcher(connector.getReceiveBufferSize());
    fetcher.start(connector.getChannel());

    LogDecoder decoder = new LogDecoder();
    decoder.handle(LogEvent.ROTATE_EVENT);
    decoder.handle(LogEvent.FORMAT_DESCRIPTION_EVENT);
    decoder.handle(LogEvent.QUERY_EVENT);
    decoder.handle(LogEvent.XID_EVENT);

    LogContext context = new LogContext();
    // 若entry position存在gtid，则使用传入的gtid作为gtidSet
    // 拼接的标准,否则同时开启gtid和tsdb时，会导致丢失gtid
    // 而当源端数据库gtid 有purged时会有如下类似报错
    // 'errno = 1236, sqlstate = HY000 errmsg = The slave is connecting
    // using CHANGE MASTER TO MASTER_AUTO_POSITION = 1 ...
    if (StringUtils.isNotEmpty(gtid)) {
        decoder.handle(LogEvent.GTID_LOG_EVENT);
        context.setGtidSet(MysqlGTIDSet.parse(gtid));
    }
    context.setFormatDescription(new FormatDescriptionLogEvent(4, binlogChecksum));

    while (fetcher.fetch()) {
        accumulateReceivedBytes(fetcher.limit());
        LogEvent event = null;
        event = decoder.decode(fetcher, context);

        if (event == null) {
            throw new CanalParseException("parse failed");
        }

        if (!func.sink(event)) {
            break;
        }
    }
}// end seek
```
### (9). MysqlConnection.updateSettings
```
private void updateSettings() throws IOException {
    try {
        update("set wait_timeout=9999999");
    } catch (Exception e) {
        logger.warn("update wait_timeout failed", e);
    }
    try {
        update("set net_write_timeout=1800");
    } catch (Exception e) {
        logger.warn("update net_write_timeout failed", e);
    }

    try {
        update("set net_read_timeout=1800");
    } catch (Exception e) {
        logger.warn("update net_read_timeout failed", e);
    }

    try {
        // 设置服务端返回结果时不做编码转化，直接按照数据库的二进制编码进行发送，由客户端自己根据需求进行编码转化
        update("set names 'binary'");
    } catch (Exception e) {
        logger.warn("update names failed", e);
    }

    try {
        // mysql5.6针对checksum支持需要设置session变量
        // 如果不设置会出现错误： Slave can not handle replication events with the
        // checksum that master is configured to log
        // 但也不能乱设置，需要和mysql server的checksum配置一致，不然RotateLogEvent会出现乱码
        // '@@global.binlog_checksum'需要去掉单引号,在mysql 5.6.29下导致master退出
        update("set @master_binlog_checksum= @@global.binlog_checksum");
    } catch (Exception e) {
        if (!StringUtils.contains(e.getMessage(), "Unknown system variable")) {
            logger.warn("update master_binlog_checksum failed", e);
        }
    }

    try {
        // 参考:https://github.com/alibaba/canal/issues/284
        // mysql5.6需要设置slave_uuid避免被server kill链接
        update("set @slave_uuid=uuid()");
    } catch (Exception e) {
        if (!StringUtils.contains(e.getMessage(), "Unknown system variable")) {
            logger.warn("update slave_uuid failed", e);
        }
    }

    try {
        // mariadb针对特殊的类型，需要设置session变量
        update("SET @mariadb_slave_capability='" + LogEvent.MARIA_SLAVE_CAPABILITY_MINE + "'");
    } catch (Exception e) {
        if (!StringUtils.contains(e.getMessage(), "Unknown system variable")) {
            logger.warn("update mariadb_slave_capability failed", e);
        }
    }

    try {
        long periodNano = TimeUnit.SECONDS.toNanos(MASTER_HEARTBEAT_PERIOD_SECONDS);
        update("SET @master_heartbeat_period=" + periodNano);
    } catch (Exception e) {
        logger.warn("update master_heartbeat_period failed", e);
    }
} // end updateSettings
```
### (10). MysqlConnection.sendBinlogDump
```
private void sendBinlogDump(String binlogfilename, Long binlogPosition) throws IOException {
    // binlogfilename = mysql-bin.000021
    // binlogPosition = 4
    // 创建binlogDump命令
    BinlogDumpCommandPacket binlogDumpCmd = new BinlogDumpCommandPacket();
    binlogDumpCmd.binlogFileName = binlogfilename;
    binlogDumpCmd.binlogPosition = binlogPosition;
    // 1779695896
    binlogDumpCmd.slaveServerId = this.slaveId;
    byte[] cmdBody = binlogDumpCmd.toBytes();

    logger.info("COM_BINLOG_DUMP with position:{}", binlogDumpCmd);
    // 创建协议头
    HeaderPacket binlogDumpHeader = new HeaderPacket();
    // 设置协议体的长度
    binlogDumpHeader.setPacketBodyLength(cmdBody.length);
    // sequenceNumber = 0
    binlogDumpHeader.setPacketSequenceNumber((byte) 0x00);
    // 向MySQL发起dump请求
    PacketManager.writePkg(connector.getChannel(), binlogDumpHeader.toBytes(), cmdBody);
    connector.setDumping(true);
}
```
### (11). UML图

!["MysqlEventParser dump过程"](/assets/canal/imgs/MysqlEventParser-uml-sequence-thread.jpg)

### (12). 总结
1. 从持久层(FailbackLogPositionManager),中获取最后一次持久的信息,包装成LogPosition并返回.
2. 如果LogPosition为空,判断当前的MysqlConnection的连接是:master还是standby,我这里是master
3. 从配置文件中获取:canal.instance.master.journal.name,该参数为用户指定的:binlogfilename    
4. 如果canal.instance.master.journal.name为空,则,查看是否有配置:canal.instance.master.timestamp,如果条件成立,则调用:findByStartTimeStamp(mysqlConnection, entryPosition.getTimestamp())方法.
5. 如果canal.instance.master.journal.name为空,canal.instance.master.timestamp配置也未指定,则调用:findEndPositionWithMasterIdAndTimestamp(mysqlConnection),从mysql最新的binlog和position.   
6. 向MySQL发送:show master status,获得MySQL的binlogname(mysql-bin.000021)和position(4)信息并转换为业务模型:EntryPosition
7. 从给定的binlogfilename(mysql-bin.000021)开始查找指定时间的position
8. 把请求委托给:MysqlConnection.seek方法,进行查找
9. MysqlConnection.updateSettings方法,更新MySqlConnection信息.    
    9.1. set wait_timeout=9999999;    
    9.2. set net_write_timeout=1800;   
    9.3. set net_read_timeout=1800;   
    9.4. set names 'binary';   
    9.5. set @master_binlog_checksum= @@global.binlog_checksum;   
    9.6. set @slave_uuid=uuid();   
    9.7. SET @mariadb_slave_capability='4';   
    9.8. SET @master_heartbeat_period=15000000000;   
10. MysqlConnection.loadBinlogChecksum  
    10.1 select @@global.binlog_checksum; // 获取binlog_checksum  
11. MysqlConnection.sendBinlogDump   
12. DirectLogFetcher的内容留到另一节讲.