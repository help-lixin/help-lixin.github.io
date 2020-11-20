---
layout: post
title: 'Canal Server源码之五(MysqlEventParser)'
date: 2019-10-01
author: 李新
tags: Canal
---

### (1). 概述

> MysqlEventParser故名思意,创建一个线程模似MySQL Slave,发送Dump协议.而Dump里的内容,MySQL称之为:Event. 

### (2). MysqlEventParser类结构图
!["MysqlEventParser类继承关系图"](/assets/canal/imgs/MysqlEventParser-uml-class.jpg)

### (3). MysqlEventParser.start
```

// 数据库信息
// 主库
protected AuthenticationInfo masterInfo;
// 备库
protected AuthenticationInfo standbyInfo;
// 运行中的数据库信息
protected volatile AuthenticationInfo runningInfo;

public void start() throws CanalParseException {
    if (runningInfo == null) { // 第一次链接主库
        // masterInfo是从实例(example)的配置文件中读取到的MySQL信息
        runningInfo = masterInfo;
    }

    // 调用父类
    super.start();
}

# masterInfo对应的xml配置
# <property name="masterInfo">
#    <bean class="com.alibaba.otter.canal.parse.support.AuthenticationInfo" init-method="initPwd">
#        <property name="address" value="${canal.instance.master.address}" />
#        <property name="username" value="${canal.instance.dbUsername:retl}" />
#        <property name="password" value="${canal.instance.dbPassword:retl}" />
#        <property name="pwdPublicKey" value="${canal.instance.pwdPublicKey:retl}" />
#        <property name="enableDruid" value="${canal.instance.enableDruid:false}" />
#        <property name="defaultDatabaseName" value="${canal.instance.defaultDatabaseName:}" />
#    </bean>
# </property>

```
### (4). AbstractMysqlEventParser.start
```
TableMetaTSDB    tableMetaTSDB;

public void start() throws CanalParseException {

    if (enableTsdb) { //true
        // 在xml有配置:enableTsdb=true,所以会触发实例化,不会进入该方法体
        if (tableMetaTSDB == null) { //false
            synchronized (CanalEventParser.class) {
                try {
                    // 设置当前正在加载的通道，加载spring查找文件时会用到该变量
                    System.setProperty("canal.instance.destination", destination);
                    // 初始化
                    tableMetaTSDB = tableMetaTSDBFactory.build(destination, tsdbSpringXml);
                } finally {
                    System.setProperty("canal.instance.destination", "");
                }
            }
        }
    }

    // 调用父类
    super.start();
}

# 启用时,同时为对象:TableMetaTSDB进行实例化
public void setEnableTsdb(boolean enableTsdb) {
    this.enableTsdb = enableTsdb;
    if (this.enableTsdb) {
        if (tableMetaTSDB == null) {
            // 初始化
            tableMetaTSDB = tableMetaTSDBFactory.build(destination, tsdbSpringXml);
        }
    }
}

# xml中配置:enableTsdb,会触发:TableMetaTSDB对象的实例化

<!--表结构相关-->
# <property name="enableTsdb" value="${canal.instance.tsdb.enable:true}"/>
# <property name="tsdbSpringXml" value="${canal.instance.tsdb.spring.xml:}"/>
# <property name="tsdbSnapshotInterval" value="${canal.instance.tsdb.snapshot.interval:24}" />
# <property name="tsdbSnapshotExpire" value="${canal.instance.tsdb.snapshot.expire:360}" />

```
### (5). AbstractEventParser.start
```

protected EventTransactionBuffer transactionBuffer;
protected int transactionSize = 1024;

protected BinlogParser  binlogParser = null;
protected Thread        parseThread  = null;

// 线程出现异常的处理
protected Thread.UncaughtExceptionHandler  handler 
                = new Thread.UncaughtExceptionHandler() {
    public void uncaughtException(Thread t,Throwable e) {
        logger.error("parse events has an error",e);
    } //end uncaughtException
};

public AbstractEventParser(){
    // 初始化一下
    transactionBuffer = new EventTransactionBuffer(new TransactionFlushCallback() {

        public void flush(List<CanalEntry.Entry> transaction) throws InterruptedException {
            boolean successed = consumeTheEventAndProfilingIfNecessary(transaction);
            if (!running) {
                return;
            }

            if (!successed) {
                throw new CanalParseException("consume failed!");
            }

            LogPosition position = buildLastTransactionPosition(transaction);
            if (position != null) { // 可能position为空
                logPositionManager.persistLogPosition(AbstractEventParser.this.destination, position);
            }
        }
    });
}//end AbstractEventParser构造器



public void start() {
    super.start();
    MDC.put("destination", destination);

    // 配置transaction buffer
    // 初始化缓冲队列
    // transactionBuffer在:AbstractEventParser的构造器中进行初始化的
    transactionBuffer.setBufferSize(transactionSize);// 设置buffer大小
    // 启动事件缓冲区
    // ************************ 6.EventTransactionBuffer.start() ********************************
    transactionBuffer.start();


    // 构造bin log parser
    // 启动事件缓冲区
    // ************************ 7.AbstractMysqlEventParser.buildParser() ********************************
    binlogParser = buildParser();// 初始化一下BinLogParser

    // ************************ 8.LogEventConvert.start()********************************
    binlogParser.start();

    // 创建解析线程
    parseThread = new Thread(new Runnable() {
        public void run() {
            // ...
        }
    }); //end new Thread

    // 为线程设置异常处理
    parseThread.setUncaughtExceptionHandler(handler);
    // 给线程设置名称:
    // destination = example , address = /127.0.0.1:3306 , EventParser
    parseThread.setName(String.format("destination = %s , address = %s , EventParser",
        destination,
        runningInfo == null ? null : runningInfo.getAddress()));

    // 启动线程 
    parseThread.start();
}// end start

```
### (6). EventTransactionBuffer.start
```
# EventTransactionBuffer是在:AbstractEventParser的构造器中创建的,以及配置:TransactionFlushCallback.

private int                      bufferSize    = 1024;
private int                      indexMask;
private CanalEntry.Entry[]       entries;

private TransactionFlushCallback flushCallback;

public void start() throws CanalStoreException {
    super.start();
    // 校验参数
    if (Integer.bitCount(bufferSize) != 1) { // false
        throw new IllegalArgumentException("bufferSize must be a power of 2");
    }

    Assert.notNull(flushCallback, "flush callback is null!");
    indexMask = bufferSize - 1;
    // 创建一个CanalEntry数组
    entries = new CanalEntry.Entry[bufferSize];
}
```
### (7). AbstractMysqlEventParser.buildParser()
```
protected BinlogParser buildParser() {
    LogEventConvert convert = new LogEventConvert();
    if (eventFilter != null && eventFilter instanceof AviaterRegexFilter) { //true
        // eventFilter = "^.*\..*$"
        convert.setNameFilter((AviaterRegexFilter) eventFilter);
    }

    if (eventBlackFilter != null && eventBlackFilter instanceof AviaterRegexFilter) { //true
         //eventBlackFilter=""
        convert.setNameBlackFilter((AviaterRegexFilter) eventBlackFilter);
    }
    
    // getFieldFilterMap() = {}
    convert.setFieldFilterMap(getFieldFilterMap());
    // getFieldBlackFilterMap() = {}
    convert.setFieldBlackFilterMap(getFieldBlackFilterMap());

    // connectionCharset = "UTF-8"
    convert.setCharset(connectionCharset);
    // filterQueryDcl = false
    convert.setFilterQueryDcl(filterQueryDcl);
    // filterQueryDml = false
    convert.setFilterQueryDml(filterQueryDml);
    // filterQueryDdl = false
    convert.setFilterQueryDdl(filterQueryDdl);
    // filterRows = false
    convert.setFilterRows(filterRows);
    // filterTableError = false
    convert.setFilterTableError(filterTableError);
    // useDruidDdlFilter = true
    convert.setUseDruidDdlFilter(useDruidDdlFilter);
    return convert;
}// end buildParser
```
### (8). LogEventConvert.start
```
public void start() {
    if (running) {
        throw new CanalException(this.getClass().getName() + " has startup , don't repeat start");
    }
    running = true;
}
```
### (9). UML图解
!["MySqlEventParser启动时序图"](/assets/canal/imgs/MysqlEventParser-uml-sequence-start.jpg)

### (10). 总结
> 1. 创建:EventTransactionBuffer,并启动(start)  
> 2. 构造:BinlogParser(LogEventConvert),并启动(start)  
> 3. 创建解析线程,并启动(start)  
> 4. 这一节,不分析线程内部的内容.