---
layout: post
title: 'Canal Server源码之五(MysqlEventParser-5)'
date: 2019-10-01
author: 李新
tags: Canal
---

### (1). 概述
> 前面内容讲解了:创建MySQL连接/创建心跳连接/为连接配置超时/查找binlog和position,这一节,开始主要讲解,执行dump过程.Canal默认是支持并行(canal.instance.parser.parallel = true)

### (2). AbstractEventParser.run
```

public void start() {
    parseThread = new Thread(new Runnable() {
        public void run() {
            while (running) {
                // 忽略其它信息
                // ... ... 
                
                // 开始dump数据
                if (parallel) {
                    // build stage processor
                    // *************************************************************
                    // 3.委托给子类:AbstractMysqlEventParser.buildMultiStageCoprocessor()
                    // *************************************************************
                    multiStageCoprocessor = buildMultiStageCoprocessor();

                    // 如果是GTID模式
                    if (isGTIDMode() 
                        && StringUtils.isNotEmpty(startPosition.getGtid())) { // false
                        // 判断所属instance是否启用GTID模式，是的话调用ErosaConnection中GTID对应方法dump数据
                        GTIDSet gtidSet = MysqlGTIDSet.parse(startPosition.getGtid());
                        ((MysqlMultiStageCoprocessor) multiStageCoprocessor).setGtidSet(gtidSet);
                        multiStageCoprocessor.start();
                        erosaConnection.dump(gtidSet, multiStageCoprocessor);
                    } else { // 非Gtid模式

                        // **********************************************************
                        // 5. 启动多阶段协同处理器(启动RingBuffer)
                        // **********************************************************
                        multiStageCoprocessor.start();

                        if (StringUtils.isEmpty(startPosition.getJournalName())
                            && startPosition.getTimestamp() != null) {  //false
                            erosaConnection.dump(startPosition.getTimestamp(), multiStageCoprocessor);
                        } else {
                            // *****************************************************
                            // 6. ErosaConnection(MysqlConnection)进行dump,然后,交给:MultiStageCoprocessor处理
                            // *****************************************************
                            erosaConnection.dump(startPosition.getJournalName(),
                                startPosition.getPosition(),
                                multiStageCoprocessor);
                        }
                    }// end else

                }// end if

                // 忽略其它信息
                // ... ... 
            }// end while
        }// end run
    }
}
```
### (3). AbstractMysqlEventParser.buildMultiStageCoprocessor
```
protected MultiStageCoprocessor buildMultiStageCoprocessor() {
    // *******************************************************************
    // 4. new MysqlMultiStageCoprocessor
    // *******************************************************************
    MysqlMultiStageCoprocessor mysqlMultiStageCoprocessor 
        = new MysqlMultiStageCoprocessor(parallelBufferSize,
            parallelThreadSize,
            (LogEventConvert) binlogParser,
            transactionBuffer,
            destination);
    //  eventsPublishBlockingTime = 0
    mysqlMultiStageCoprocessor.setEventsPublishBlockingTime(eventsPublishBlockingTime);
    return mysqlMultiStageCoprocessor;
}
```
### (4). MysqlMultiStageCoprocessor构造器
```
public MysqlMultiStageCoprocessor(
            //  256
            int ringBufferSize, 
            // 2
            int parserThreadCount, 
            // com.alibaba.otter.canal.parse.inbound.mysql.dbsync.LogEventConvert
            LogEventConvert logEventConvert,
            // com.alibaba.otter.canal.parse.inbound.EventTransactionBuffer
            EventTransactionBuffer transactionBuffer, 
            // example
            String destination){
                this.ringBufferSize = ringBufferSize;
                this.parserThreadCount = parserThreadCount;
                this.logEventConvert = logEventConvert;
                this.transactionBuffer = transactionBuffer;
                this.destination = destination;
}
```
### (5). MysqlMultiStageCoprocessor.start

```
// 事件流转图如下:先串行,再并行,后再串行合并.
// 有疑问的是:针对同一张表的并行解析.要如何解决呢?
//                            -> DmlParserStage
//  -> SimpleParserStage                             -> SinkStoreStage
//                            -> DmlParserStage
public void start() {
    super.start();
    this.exception = null;
    this.disruptorMsgBuffer = RingBuffer.createSingleProducer(
        new MessageEventFactory(),
        ringBufferSize,
        new BlockingWaitStrategy());

    int tc = parserThreadCount > 0 ? parserThreadCount : 1;

    this.parserExecutor = Executors.newFixedThreadPool(tc, new NamedThreadFactory("MultiStageCoprocessor-Parser-" + destination));

    this.stageExecutor = Executors.newFixedThreadPool(2, new NamedThreadFactory("MultiStageCoprocessor-other-" + destination));

    SequenceBarrier sequenceBarrier = disruptorMsgBuffer.newBarrier();
    ExceptionHandler exceptionHandler = new SimpleFatalExceptionHandler();

    // stage 2
    this.logContext = new LogContext();
    simpleParserStage = new BatchEventProcessor<MessageEvent>(disruptorMsgBuffer,
        sequenceBarrier,
        new SimpleParserStage(logContext));
    simpleParserStage.setExceptionHandler(exceptionHandler);
    disruptorMsgBuffer.addGatingSequences(simpleParserStage.getSequence());

    // stage 3
    SequenceBarrier dmlParserSequenceBarrier = disruptorMsgBuffer.newBarrier(simpleParserStage.getSequence());

    WorkHandler<MessageEvent>[] workHandlers = new DmlParserStage[tc];
    for (int i = 0; i < tc; i++) {
        workHandlers[i] = new DmlParserStage();
    }

    workerPool = new WorkerPool<MessageEvent>(disruptorMsgBuffer,
        dmlParserSequenceBarrier,
        exceptionHandler,
        workHandlers);
    Sequence[] sequence = workerPool.getWorkerSequences();
    disruptorMsgBuffer.addGatingSequences(sequence);

    // stage 4
    SequenceBarrier sinkSequenceBarrier = disruptorMsgBuffer.newBarrier(sequence);
    sinkStoreStage = new BatchEventProcessor<MessageEvent>(
        disruptorMsgBuffer,
        sinkSequenceBarrier,
        new SinkStoreStage());
    sinkStoreStage.setExceptionHandler(exceptionHandler);
    disruptorMsgBuffer.addGatingSequences(sinkStoreStage.getSequence());

    // start work
    stageExecutor.submit(simpleParserStage);
    stageExecutor.submit(sinkStoreStage);


    workerPool.start(parserExecutor);
}// end start
```
### (6). MysqlConnection.dump
```
public void dump(
        // binlogfile = mysql-bin.000022
        String binlogfilename, 
        // binlog position = 4
        Long binlogPosition, 
        // 回调处理函数(MysqlMultiStageCoprocessor)
        MultiStageCoprocessor coprocessor) throws IOException {

    /**
    *  set wait_timeout=9999999;
    *  set net_write_timeout=1800;
    *  set net_read_timeout=1800;
    *  set names 'binary';
    *  set @master_binlog_checksum= @@global.binlog_checksum;
    *  set @slave_uuid=uuid();
    *  SET @mariadb_slave_capability=4;
    *  SET @master_heartbeat_period=15;
    */
    updateSettings();

    /**
    * 获取binlog_format
    * show variables like 'binlog_format'
    */
    loadBinlogChecksum();

    // *****************************************************************
    // 7.注册Slave,并对协议返回结果进行解析
    // *****************************************************************
    sendRegisterSlave();

    // 发送binlog dump请求,把解析过程委托给:DirectLogFetcher
    sendBinlogDump(binlogfilename, binlogPosition);
    ((MysqlMultiStageCoprocessor) coprocessor).setConnection(this);
    ((MysqlMultiStageCoprocessor) coprocessor).setBinlogChecksum(binlogChecksum);

    // 从connection中拉取数据,并调用:回调函数的:publish
    DirectLogFetcher fetcher = new DirectLogFetcher(connector.getReceiveBufferSize());
    try {
        fetcher.start(connector.getChannel());
        while (fetcher.fetch()) {
            accumulateReceivedBytes(fetcher.limit());
            // fetcher结果存储在内部的数组里
            LogBuffer buffer = fetcher.duplicate();
            // 告诉fetch已经消费了多少个字节
            fetcher.consume(fetcher.limit());
            // ********************************************************************
            // 委托:MultiStageCoprocessor.publish(xxxx),直到返回:false则退出循环.
            // ********************************************************************
            if (!coprocessor.publish(buffer)) {
                break;
            }
        }
    } finally {
        fetcher.close();
    }
}// end dump
```
### (7). MysqlConnection.sendRegisterSlave
```
private void sendRegisterSlave() throws IOException {
    
    // 创建注册Slave命令
    RegisterSlaveCommandPacket cmd = new RegisterSlaveCommandPacket();
    SocketAddress socketAddress = connector.getChannel().getLocalSocketAddress();
    if (socketAddress == null || !(socketAddress instanceof InetSocketAddress)) {
        return;
    }

    InetSocketAddress address = (InetSocketAddress) socketAddress;
    String host = address.getHostString();
    int port = address.getPort();
    cmd.reportHost = host;
    cmd.reportPort = port;
    cmd.reportPasswd = authInfo.getPassword();
    cmd.reportUser = authInfo.getUsername();
    cmd.serverId = this.slaveId;
    
    byte[] cmdBody = cmd.toBytes();

    logger.info("Register slave {}", cmd);

    // 创建协议头信息(指定:协议头长度和sequenceNumber)
    HeaderPacket header = new HeaderPacket();
    header.setPacketBodyLength(cmdBody.length);
    header.setPacketSequenceNumber((byte) 0x00);
    
    // 发送协议
    PacketManager.writePkg(connector.getChannel(), header.toBytes(), cmdBody);

    // 解析返回协议头
    header = PacketManager.readHeader(connector.getChannel(), 4);
    // 根据协议头信息,获得协议体
    byte[] body = PacketManager.readBytes(connector.getChannel(), header.getPacketBodyLength());
    // 协议体不能为空
    assert body != null;
    // 协议体的第一个字节为: 小于零 或者  等于负一 则抛出异常
    if (body[0] < 0) {  // false
        if (body[0] == -1) {
            ErrorPacket err = new ErrorPacket();
            err.fromBytes(body);
            throw new IOException("Error When doing Register slave:" + err.toString());
        } else {
            throw new IOException("unpexpected packet with field_count=" + body[0]);
        }
    }
}// end sendRegisterSlave
```
### (8). UML图
!["Canal 进行dump UML图"](/assets/canal/imgs/MysqlEventParser-dump.jpg)

### (9). 总结
> 1. 首先判断是否为并行解析(anal.instance.parser.parallel=true)  
> 2. <font color='red'>构建多阶段解析器类:MysqlMultiStageCoprocessor(SimpleParserStage/DmlParserStage/SinkStoreStage)</font>   
> 3. <font color='red'>调用:MysqlConnection.dump方法,并传递回调函数为:MysqlMultiStageCoprocessor</font>   
> 4. 调用:MysqlConnection.sendRegisterSlave注册slave信息.   
> 5. 调用:MysqlConnection.sendBinlogDump要求接受dump信息.   
> 6. 创建:DirectLogFetcher,它主要用于从:MysqlConnection返回流中获取数据.   
> 7. 如果while(fetch.fetch())为:true,则读取流中的信息,并转换为:LogBuffer.  
> 8. <font color='red'>回调第(3)步的回调函数:MysqlMultiStageCoprocessor.publish(LogBuffer).</font>  