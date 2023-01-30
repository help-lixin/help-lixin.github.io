---
layout: post
title: 'RocketMQ Broker消息存储(二)' 
date: 2019-12-10
author: 李新
tags:  RocketMQ
---

### (1). 概述
前面把生产者和消费者都撸了遍源码,这一篇,主要介绍Broker,由于,Broker依赖于网络调用,所以,会比较麻烦,而RocketMQ写了大量的测试类,我们可以可以直接使用测试类来帮且我们辅助学习. 

### (2). DefaultMessageStoreTest
```
public class DefaultMessageStoreTest {
    private final String StoreMessage = "Once, there was a chance for me!";
    private int QUEUE_TOTAL = 100;
    private AtomicInteger QueueId = new AtomicInteger(0);
    private SocketAddress BornHost;
    private SocketAddress StoreHost;
    private byte[] MessageBody;
    private MessageStore messageStore;
    
    @Before
    public void init() throws Exception {
        StoreHost = new InetSocketAddress(InetAddress.getLocalHost(), 8123);
        BornHost = new InetSocketAddress(InetAddress.getByName("127.0.0.1"), 0);
        // 1. 构建消息存储对象(MessageStore )
        messageStore = buildMessageStore();
        // 4. 启动
        boolean load = messageStore.load();
        assertTrue(load);
        // 6. 启动
        messageStore.start();
    }
}
```
### (3). 构建DefaultMessageStore(DefaultMessageStoreTest)
```
private MessageStore buildMessageStore() throws Exception {
        MessageStoreConfig messageStoreConfig = new MessageStoreConfig();
        // 单个conmmitlog文件大小默认1GB
        messageStoreConfig.setMapedFileSizeCommitLog(1024 * 1024 * 10);
        // 单个consumequeue文件大小默认30W*20表示单个Consumequeue文件中存储30W个ConsumeQueue
        messageStoreConfig.setMapedFileSizeConsumeQueue(1024 * 1024 * 10);
        // 单个索引文件hash槽的个数,默认为五百万
        messageStoreConfig.setMaxHashSlotNum(10000);
        // 单个索引文件索引条目的个数,默认为两千万
        messageStoreConfig.setMaxIndexNum(100 * 100);
        // 同步刷盘
        messageStoreConfig.setFlushDiskType(FlushDiskType.SYNC_FLUSH);
        // consumuQueue文件刷盘频率(TimeUnit.MILLISECONDS)
        messageStoreConfig.setFlushIntervalConsumeQueue(1);
       // 2.创建:DefaultMessageStore
       return new DefaultMessageStore(                   //
            messageStoreConfig,                         // 消息存储配置 
            new BrokerStatsManager("simpleTest"),       // 统计管理配置 
            new MyMessageArrivingListener(),            // 消息保存成功后的回调
            new BrokerConfig()                          // Brokder配置信息
        );
}

// 消息刷盘成功之后回调地址.可实现消息的推送或者拉取(PullRequestHoldService)
private class MyMessageArrivingListener 
        implements MessageArrivingListener {
    @Override
    public void arriving(
        String topic, 
        int queueId, 
        long logicOffset, 
        long tagsCode, 
        long msgStoreTime,
        byte[] filterBitMap, 
        Map<String, String> properties) {
            
    }
}
```
### (4). DefaultMessageStore
```
package org.apache.rocketmq.store;

public class DefaultMessageStore 
      implements MessageStore {
    // 消息存储的配置管理      
    private final MessageStoreConfig messageStoreConfig;
    // commitlog日志管理
    private final CommitLog commitLog;
    
    // 主题对应的队列集合
    private final ConcurrentMap<
         String/* topic */, 
         ConcurrentMap<Integer/* queueId */, ConsumeQueue>> consumeQueueTable;
    
    // consumequeue刷盘管理
    private final FlushConsumeQueueService flushConsumeQueueService;
    // 清除commitlog管理
    private final CleanCommitLogService cleanCommitLogService;
    // 清除consumerqueue管理
    private final CleanConsumeQueueService cleanConsumeQueueService;
    // 存储统计管理
    private final StoreStatsService storeStatsService;
    
         
   // 消息刷盘之后回调(Brokder角色非SLAVE,并且:longPollingEnable=true)
   // 才会通知 
   private final MessageArrivingListener messageArrivingListener;
   // Brokder配置管理
   private final BrokerConfig brokerConfig;
   // Brokder统计管理
   private final BrokerStatsManager brokerStatsManager;
   // MappendFile管理
   private final AllocateMappedFileService allocateMappedFileService;
   // consumerqueue刷盘
   private final FlushConsumeQueueService flushConsumeQueueService;
   // index文件管理
   private final IndexService indexService;
   // 消息刷盘控制(将会控制index/consumerqueue刷盘)
   private final ReputMessageService reputMessageService;
   // 高可用管理
   private final HAService haService;
   // 定时消息管理
   private final ScheduleMessageService scheduleMessageService;
   // 事务消息存储池
   private final TransientStorePool transientStorePool;
   // commitlog刷盘之后分发器
   private final LinkedList<CommitLogDispatcher> dispatcherList;
   // 加载文件
   private RandomAccessFile lockFile;
          
    // 3. DefaultMessageStore构造器
    public DefaultMessageStore(
            final MessageStoreConfig messageStoreConfig, 
            final BrokerStatsManager brokerStatsManager,
           final MessageArrivingListener messageArrivingListener, 
           final BrokerConfig brokerConfig ) throws IOException {
               
        this.messageArrivingListener = messageArrivingListener;
        this.brokerConfig = brokerConfig;
        this.messageStoreConfig = messageStoreConfig;
        this.brokerStatsManager = brokerStatsManager;
        // 创建MappendFile管理器(专门负责创建commitlog文件)
        this.allocateMappedFileService = new AllocateMappedFileService(this);
        // commitlog管理
        this.commitLog = new CommitLog(this);
        // 主题与队列关系映射
        this.consumeQueueTable = new ConcurrentHashMap<>(32);
        // consumerQueue刷盘管理
        this.flushConsumeQueueService = new FlushConsumeQueueService();
        // 清除commitLog管理
        this.cleanCommitLogService = new CleanCommitLogService();
        // 清除consumequeue服务
        this.cleanConsumeQueueService = new CleanConsumeQueueService();
        // 存储统计
        this.storeStatsService = new StoreStatsService();
        // index文件管理
        this.indexService = new IndexService(this);
        // 高可用管理
        this.haService = new HAService(this);
        // commit消息分发器,根据CommitLog文件构建ConsumeQueue/IndexFile
        this.reputMessageService = new ReputMessageService();
        // 定时任务消息管理
        this.scheduleMessageService = new ScheduleMessageService(this);
        // 事务消息存储池(*),以后再细看
        this.transientStorePool = new TransientStorePool(messageStoreConfig);
        if (messageStoreConfig.isTransientStorePoolEnable()) {
            this.transientStorePool.init();
        }
        // MappedFile启动
        this.allocateMappedFileService.start();
        // IndexFile线程启动
        this.indexService.start();
        
        // CommitLog文件转发处理
        this.dispatcherList = new LinkedList<>();
        this.dispatcherList.addLast(new CommitLogDispatcherBuildConsumeQueue());
        this.dispatcherList.addLast(new CommitLogDispatcherBuildIndex());
        // C:\Users\lixin\store\lock
        File file = new File(StorePathConfigHelper.getLockFile(messageStoreConfig.getStorePathRootDir()));
       //  确保(创建) C:\Users\lixin\store 存在
       MappedFile.ensureDirOK(file.getParent());
       // C:\Users\lixin\store\lock 创建
       lockFile = new RandomAccessFile(file, "rw");
    }   
}
```
### (5). DefaultMessageStore
```
package org.apache.rocketmq.store;
public class DefaultMessageStore 
       implements MessageStore {
    // 5. load       
    public boolean load() {
        boolean result = true;
        try {
            // 查看abort文件是否存在
            boolean lastExitOK = !this.isTempFileExist();
            // true:代表正常关闭
            // false:非正常关闭
            log.info("last shutdown {}", lastExitOK ? "normally" : "abnormally");
            
            // 定时消息管理服务启动
            if (null != scheduleMessageService) {
                // delayOffset.json
                // 1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h
                
                result = result && this.scheduleMessageService.load();
            }

            // load Commit Log
            // C:\Users\lixin\store\commitlog
            result = result && this.commitLog.load();
            
            // load Consume Queue
            // C:\Users\lixin\store\consumequeue
            result = result && this.loadConsumeQueue();

            if (result) {
                // 创建checkpoint文件
                // C:\Users\lixin\store\checkpoint
                // MappedFile.OS_PAGE_SIZE = 4096
                // fileChannel.map(MapMode.READ_WRITE, 0, MappedFile.OS_PAGE_SIZE);
                // 物理消息时间
                // this.mappedByteBuffer.putLong(0, this.physicMsgTimestamp);
                // 逻辑消息时间
                // this.mappedByteBuffer.putLong(8, this.logicsMsgTimestamp);
                // indexMsg消息时间
                // this.mappedByteBuffer.putLong(16, this.indexMsgTimestamp);                
                this.storeCheckpoint =
                    new StoreCheckpoint(StorePathConfigHelper.getStoreCheckpoint(this.messageStoreConfig.getStorePathRootDir()));
                
                // 索引文件
                // C:\Users\lixin\store\index
                this.indexService.load(lastExitOK);
                
                // 恢复(true)
                this.recover(lastExitOK);
                log.info("load over, and the max phy offset = {}", this.getMaxPhyOffset());
            }
        } catch (Exception e) {
            log.error("load exception", e);
            result = false;
        }
        
        // 启动没有成功的情况下,关闭MappedFile
        if (!result) {
            this.allocateMappedFileService.shutdown();
        }
        return result;
    } //end load
    
    
    private void recover(final boolean lastExitOK) {
        // 恢复consumequeue
        this.recoverConsumeQueue();
        if (lastExitOK) {
            // 正常恢复comitlog
            this.commitLog.recoverNormally();
        } else {
            this.commitLog.recoverAbnormally();
        }
        // 恢复 topicqueue
        this.recoverTopicQueueTable();
    }
    
    
    private boolean isTempFileExist() {
        // C:\Users\lixin\store\abort
        String fileName = StorePathConfigHelper.getAbortFile(this.messageStoreConfig.getStorePathRootDir());
        File file = new File(fileName);
        return file.exists();
    } //end isTempFileExist
    
}
```
### (6). DefaultMessageStore
```
package org.apache.rocketmq.store;

public class DefaultMessageStore 
       implements MessageStore {
    // 6. start
    public void start() throws Exception {

        lock = lockFile.getChannel().tryLock(0, 1, false);
        if (lock == null || lock.isShared() || !lock.isValid()) {
            throw new RuntimeException("Lock failed,MQ already started");
        }
        
        // C:\Users\lixin\store\lock 写入文件
        lockFile.getChannel().write(ByteBuffer.wrap("lock".getBytes()));
        lockFile.getChannel().force(true);
        
        // consumequeue 刷盘
        this.flushConsumeQueueService.start();
        // commit刷盘
        this.commitLog.start();
        // store 统计
        this.storeStatsService.start();
        // 
        if (this.scheduleMessageService != null && 
            SLAVE != messageStoreConfig.getBrokerRole()) {
            // 定时任务的消息
            // C:\Users\lixin\store\config\delayOffset.json
            this.scheduleMessageService.start();
        }

        if (this.getMessageStoreConfig().isDuplicationEnable()) {
            this.reputMessageService.setReputFromOffset(this.commitLog.getConfirmOffset());
        } else {
            // commitLog.getMaxOffset() = 0
            this.reputMessageService.setReputFromOffset(this.commitLog.getMaxOffset());
        }
        // consumequeue/index刷盘线程
        this.reputMessageService.start();
        // 高可用(*)
        this.haService.start();
        
        // 创建:C:\Users\lixin\store\abort
        this.createTempFile();
        // 添加定时任务
        this.addScheduleTask();
        this.shutdown = false;
    } //end start
    
    
    private void addScheduleTask() {
        // messageStoreConfig.getCleanResourceInterval() = 10000
        // 每隔10秒执行一次清除文件
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
            @Override
            public void run() {
                DefaultMessageStore.this.cleanFilesPeriodically();
            }
        }, 1000 * 60, this.messageStoreConfig.getCleanResourceInterval(), TimeUnit.MILLISECONDS);

        // 10分钟
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
            @Override
            public void run() {
                DefaultMessageStore.this.checkSelf();
            }
        }, 1, 10, TimeUnit.MINUTES);

        // 1秒钟(针对执行超过1秒以上的线程打印出线程信息)
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
            @Override
            public void run() {
                if (DefaultMessageStore.this.getMessageStoreConfig().isDebugLockEnable()) {
                    try {
                        if (DefaultMessageStore.this.commitLog.getBeginTimeInLock() != 0) {
                            long lockTime = System.currentTimeMillis() - DefaultMessageStore.this.commitLog.getBeginTimeInLock();
                            if (lockTime > 1000 && lockTime < 10000000) {

                                String stack = UtilAll.jstack();
                                final String fileName = System.getProperty("user.home") + File.separator + "debug/lock/stack-"
                                    + DefaultMessageStore.this.commitLog.getBeginTimeInLock() + "-" + lockTime;
                                MixAll.string2FileNotSafe(stack, fileName);
                            }
                        }
                    } catch (Exception e) {
                    }
                }
            }
        }, 1, 1, TimeUnit.SECONDS);

    } //end addScheduleTask
    
}
```
### (7). DefaultMessageStoreTest
```
package org.apache.rocketmq.store;
public class DefaultMessageStoreTest {
    
    @Test
    public void testWriteAndRead() {
        long totalMsgs = 10;
        QUEUE_TOTAL = 1;
        MessageBody = StoreMessage.getBytes();
        for (long i = 0; i < totalMsgs; i++) {
            // 7.创建消息
            MessageExtBrokerInner  msg = buildMessage();
            // 8.保存
            messageStore.putMessage(msg);
        }
        
        for (long i = 0; i < totalMsgs; i++) {
            GetMessageResult result = messageStore.getMessage("GROUP_A", "TOPIC_A", 0, i, 1024 * 1024, null);
            assertThat(result).isNotNull();
            result.release();
        }        
    } //end testWriteAndRead
    
    // 7. 创建一条消息
    private MessageExtBrokerInner buildMessage() {
        MessageExtBrokerInner msg = new MessageExtBrokerInner();
        msg.setTopic("FooBar");
        msg.setTags("TAG1");
        msg.setKeys("Hello");
        msg.setBody(MessageBody);
        msg.setKeys(String.valueOf(System.currentTimeMillis()));
        msg.setQueueId(Math.abs(QueueId.getAndIncrement()) % QUEUE_TOTAL);
        msg.setSysFlag(0);
        msg.setBornTimestamp(System.currentTimeMillis());
        msg.setStoreHost(StoreHost);
        msg.setBornHost(BornHost);
        return msg;
    } //end buildMessage
    
}    
```
### (8). DefaultMessageStore
```
package org.apache.rocketmq.store;
public class DefaultMessageStore 
       implements MessageStore {
    
    // 8.0 保存消息
    public PutMessageResult putMessage(MessageExtBrokerInner msg) {
        if (this.shutdown) {
            // 服务不可用
            log.warn("message store has shutdown, so putMessage is forbidden");
            return new PutMessageResult(PutMessageStatus.SERVICE_NOT_AVAILABLE, null);
        }
        
        // SLAVE角色保存消息失败
        if (BrokerRole.SLAVE == this.messageStoreConfig.getBrokerRole()) {
            long value = this.printTimes.getAndIncrement();
            if ((value % 50000) == 0) {
                // 消息存储模式为:SLAVE,所以保存消息失败
                log.warn("message store is slave mode, so putMessage is forbidden ");
            }
            return new PutMessageResult(PutMessageStatus.SERVICE_NOT_AVAILABLE, null);
        }

        // 判断消息是否可写
        if (!this.runningFlags.isWriteable()) {
            long value = this.printTimes.getAndIncrement();
            if ((value % 50000) == 0) {
                log.warn("message store is not writeable, so putMessage is forbidden " + this.runningFlags.getFlagBits());
            }
            return new PutMessageResult(PutMessageStatus.SERVICE_NOT_AVAILABLE, null);
        } else {
            // 设置打印时间
            this.printTimes.set(0);
        }

        // 消息的主题(Topic)的长度不能大于:127字节
        if (msg.getTopic().length() > Byte.MAX_VALUE) {
            log.warn("putMessage message topic length too long " + msg.getTopic().length());
            return new PutMessageResult(PutMessageStatus.MESSAGE_ILLEGAL, null);
        }

        // 消息的属性不能超过127字节
        if (msg.getPropertiesString() != null && 
            msg.getPropertiesString().length() > Short.MAX_VALUE) {
            log.warn("putMessage message properties length too long " + msg.getPropertiesString().length());
            return new PutMessageResult(PutMessageStatus.PROPERTIES_SIZE_EXCEEDED, null);
        }
        // 如果pagechage繁忙抛出错误
        if (this.isOSPageCacheBusy()) {
            return new PutMessageResult(PutMessageStatus.OS_PAGECACHE_BUSY, null);
        }
        
        long beginTime = this.getSystemClock().now();
        // **********************************************
        // 开始提交消息
        // 9.  提交消息
        PutMessageResult result = this.commitLog.putMessage(msg);

        long eclipseTime = this.getSystemClock().now() - beginTime;
        if (eclipseTime > 500) {
            log.warn("putMessage not in lock eclipse time(ms)={}, bodyLength={}", eclipseTime, msg.getBody().length);
        }
        this.storeStatsService.setPutMessageEntireTimeMax(eclipseTime);

        if (null == result || !result.isOk()) {
            this.storeStatsService.getPutMessageFailedTimes().incrementAndGet();
        }

        return result;
    }   
    
    
    public boolean isOSPageCacheBusy() {
        // commitlog上一次开始写入的时间锁
        long begin = this.getCommitLog().getBeginTimeInLock();
        // 当前时间 - 上一次时间
        // diff  = 1575533625764
        long diff = this.systemClock.now() - begin;
        // messageStoreConfig.getOsPageCacheBusyTimeOutMills() = 1000
        
        // 两者都满足(代表pagechage有问题)
        return 
          diff < 10000000  && 
          diff > this.messageStoreConfig.getOsPageCacheBusyTimeOutMills();
    } //end isOSPageCacheBusy
}
```
### (9). CommitLog
```
package org.apache.rocketmq.store;
public class CommitLog {
    // 追回消息的回调
    private final AppendMessageCallback appendMessageCallback;
    
    // 构造器
    public CommitLog(final DefaultMessageStore defaultMessageStore) {
        this.mappedFileQueue = new MappedFileQueue(
          // C:\Users\lixin\store\commitlog
          defaultMessageStore.getMessageStoreConfig().getStorePathCommitLog(),
          // 10485760
          // 前面调用:setMapedFileSizeConsumeQueue设置的
          defaultMessageStore.getMessageStoreConfig().getMapedFileSizeCommitLog(), 
          // org.apache.rocketmq.store.AllocateMappedFileService
         defaultMessageStore.getAllocateMappedFileService() 
        );
        this.defaultMessageStore = defaultMessageStore;

        if (FlushDiskType.SYNC_FLUSH == defaultMessageStore.getMessageStoreConfig().getFlushDiskType()) {
            // 同步刷盘策略线程
            this.flushCommitLogService = new GroupCommitService();
        } else {
            this.flushCommitLogService = new FlushRealTimeService();
        }
        // 
        this.commitLogService = new CommitRealTimeService();
        // 追回消息回调
        this.appendMessageCallback = new DefaultAppendMessageCallback(defaultMessageStore.getMessageStoreConfig().getMaxMessageSize());
        batchEncoderThreadLocal = new ThreadLocal<MessageExtBatchEncoder>() {
            @Override
            protected MessageExtBatchEncoder initialValue() {
                return new MessageExtBatchEncoder(defaultMessageStore.getMessageStoreConfig().getMaxMessageSize());
            }
        };
        this.putMessageLock = defaultMessageStore.getMessageStoreConfig().isUseReentrantLockWhenPutMessage() ? new PutMessageReentrantLock() : new PutMessageSpinLock();
    }
    
    // 10. 提交消息
    public PutMessageResult putMessage(final MessageExtBrokerInner msg) {
        // Set the storage time
        // 设置消息的存储时间
        msg.setStoreTimestamp(System.currentTimeMillis());
        // 设置消息Body crc32
        // Set the message body BODY CRC (consider the most appropriate setting
        // on the client)
        msg.setBodyCRC(UtilAll.crc32(msg.getBody()));
        // Back to Results
        AppendMessageResult result = null;
        // 存储状态管理服务
        StoreStatsService storeStatsService = this.defaultMessageStore.getStoreStatsService();

        String topic = msg.getTopic();
        int queueId = msg.getQueueId();

        final int tranType = MessageSysFlag.getTransactionValue(msg.getSysFlag());
        if (tranType == MessageSysFlag.TRANSACTION_NOT_TYPE
            || tranType == MessageSysFlag.TRANSACTION_COMMIT_TYPE) {
            // Delay Delivery
            // 延迟消息处理
            if (msg.getDelayTimeLevel() > 0) {
                if (msg.getDelayTimeLevel() > this.defaultMessageStore.getScheduleMessageService().getMaxDelayLevel()) {
                    msg.setDelayTimeLevel(this.defaultMessageStore.getScheduleMessageService().getMaxDelayLevel());
                }
                topic = ScheduleMessageService.SCHEDULE_TOPIC;
                queueId = ScheduleMessageService.delayLevel2QueueId(msg.getDelayTimeLevel());

                // Backup real topic, queueId
                MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_TOPIC, msg.getTopic());
                MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_QUEUE_ID, String.valueOf(msg.getQueueId()));
                msg.setPropertiesString(MessageDecoder.messageProperties2String(msg.getProperties()));

                msg.setTopic(topic);
                msg.setQueueId(queueId);
            }
        }
        
        long eclipseTimeInLock = 0;
        MappedFile unlockMappedFile = null;
        // 第一次文件还未创建,mappedFile=null
        MappedFile mappedFile = this.mappedFileQueue.getLastMappedFile();

        // 加锁
        putMessageLock.lock(); //spin or ReentrantLock ,depending on store config
        try {
            // 1575622465960
            long beginLockTimestamp = this.defaultMessageStore.getSystemClock().now();
            // 1575622465960 开始任务之前加锁
            this.beginTimeInLock = beginLockTimestamp;

            // Here settings are stored timestamp, in order to ensure an orderly
            // global
            msg.setStoreTimestamp(beginLockTimestamp);

            if (null == mappedFile || mappedFile.isFull()) {
                
                // 如果缓存中不存在文件,则创建文件
                //******************************************************
                // 11. 创建commit文件,在创建文件时,计算出下一个文件大小
                mappedFile = this.mappedFileQueue.getLastMappedFile(0); // Mark: NewFile may be cause noise
            }
            
            if (null == mappedFile) {
                log.error("create mapped file1 error, topic: " + msg.getTopic() + " clientAddr: " + msg.getBornHostString());
                beginTimeInLock = 0;
                return new PutMessageResult(PutMessageStatus.CREATE_MAPEDFILE_FAILED, null);
            }

            // 18. 往MappedFile中追回消息
            result = mappedFile.appendMessage(msg, this.appendMessageCallback);
            // 21. 判断消息是否写入成功
            switch (result.getStatus()) {
                case PUT_OK:
                    // 21.1 
                    break;
                case END_OF_FILE:
                    unlockMappedFile = mappedFile;
                    // Create a new file, re-write the message
                    mappedFile = this.mappedFileQueue.getLastMappedFile(0);
                    if (null == mappedFile) {
                        // XXX: warn and notify me
                        log.error("create mapped file2 error, topic: " + msg.getTopic() + " clientAddr: " + msg.getBornHostString());
                        beginTimeInLock = 0;
                        return new PutMessageResult(PutMessageStatus.CREATE_MAPEDFILE_FAILED, result);
                    }
                    result = mappedFile.appendMessage(msg, this.appendMessageCallback);
                    break;
                case MESSAGE_SIZE_EXCEEDED:
                case PROPERTIES_SIZE_EXCEEDED:
                    beginTimeInLock = 0;
                    return new PutMessageResult(PutMessageStatus.MESSAGE_ILLEGAL, result);
                case UNKNOWN_ERROR:
                    beginTimeInLock = 0;
                    return new PutMessageResult(PutMessageStatus.UNKNOWN_ERROR, result);
                default:
                    beginTimeInLock = 0;
                    return new PutMessageResult(PutMessageStatus.UNKNOWN_ERROR, result);
            }
            eclipseTimeInLock = this.defaultMessageStore.getSystemClock().now() - beginLockTimestamp;
            beginTimeInLock = 0;
        } finally {
            // 21.2 释放锁
            putMessageLock.unlock();
        }

        if (eclipseTimeInLock > 500) {
            log.warn("[NOTIFYME]putMessage in lock cost time(ms)={}, bodyLength={} AppendMessageResult={}", eclipseTimeInLock, msg.getBody().length, result);
        }
        
        // false
        if (null != unlockMappedFile && this.defaultMessageStore.getMessageStoreConfig().isWarmMapedFileEnable()) {
            this.defaultMessageStore.unlockMappedFile(unlockMappedFile);
        }
        
        // 包裹追加消息的返回结果
        PutMessageResult putMessageResult = new PutMessageResult(PutMessageStatus.PUT_OK, result);
    
        // 统计信息(主题信息自增/主题写入了多少字节)
        // Statistics
        storeStatsService.getSinglePutMessageTopicTimesTotal(msg.getTopic()).incrementAndGet();
        storeStatsService.getSinglePutMessageTopicSizeTotal(topic).addAndGet(result.getWroteBytes());

        // 22. 刷盘
        handleDiskFlush(result, putMessageResult, msg);
        handleHA(result, putMessageResult, msg);

        return putMessageResult;
    }
    
    // 22 刷盘
    public void handleDiskFlush(AppendMessageResult result, PutMessageResult putMessageResult, MessageExt messageExt) {
        // 同步刷盘
        // Synchronization flush
        // defaultMessageStore.getMessageStoreConfig().getFlushDiskType() = SYNC_FLUSH
        if (FlushDiskType.SYNC_FLUSH == this.defaultMessageStore.getMessageStoreConfig().getFlushDiskType()) {
            // org.apache.rocketmq.store.CommitLog$GroupCommitService
            final GroupCommitService service = (GroupCommitService) this.flushCommitLogService;
            // true
            if (messageExt.isWaitStoreMsgOK()) {
                // 22.1 创建CommitRequest请求   
                GroupCommitRequest request = new GroupCommitRequest(result.getWroteOffset() + result.getWroteBytes());
                service.putRequest(request);
                // ********************************刷盘**********************************************
                boolean flushOK = request.waitForFlush(this.defaultMessageStore.getMessageStoreConfig().getSyncFlushTimeout());
                if (!flushOK) {
                    log.error("do groupcommit, wait for flush failed, topic: " + messageExt.getTopic() + " tags: " + messageExt.getTags()
                        + " client address: " + messageExt.getBornHostString());
                    putMessageResult.setPutMessageStatus(PutMessageStatus.FLUSH_DISK_TIMEOUT);
                }
            } else {
                service.wakeup();
            }
        }
        // Asynchronous flush
        else {
            if (!this.defaultMessageStore.getMessageStoreConfig().isTransientStorePoolEnable()) {
                flushCommitLogService.wakeup();
            } else {
                commitLogService.wakeup();
            }
        }
    }


    // 22.2 创建Commit服务
    class GroupCommitService extends FlushCommitLogService {
        private volatile List<GroupCommitRequest> requestsWrite = new ArrayList<GroupCommitRequest>();
        private volatile List<GroupCommitRequest> requestsRead = new ArrayList<GroupCommitRequest>();

       // 22.2 添加Request请求
        public synchronized void putRequest(final GroupCommitRequest request) {
            synchronized (this.requestsWrite) {
                this.requestsWrite.add(request);
            }
            if (hasNotified.compareAndSet(false, true)) {
                waitPoint.countDown(); // notify
            }
        }

        private void swapRequests() {
            List<GroupCommitRequest> tmp = this.requestsWrite;
            this.requestsWrite = this.requestsRead;
            this.requestsRead = tmp;
        }

        private void doCommit() {
            synchronized (this.requestsRead) {
                if (!this.requestsRead.isEmpty()) {
                    for (GroupCommitRequest req : this.requestsRead) {
                        // There may be a message in the next file, so a maximum of
                        // two times the flush
                        boolean flushOK = false;
                        for (int i = 0; i < 2 && !flushOK; i++) {
                            flushOK = CommitLog.this.mappedFileQueue.getFlushedWhere() >= req.getNextOffset();

                            if (!flushOK) {
                                CommitLog.this.mappedFileQueue.flush(0);
                            }
                        }

                        req.wakeupCustomer(flushOK);
                    }

                    long storeTimestamp = CommitLog.this.mappedFileQueue.getStoreTimestamp();
                    if (storeTimestamp > 0) {
                        CommitLog.this.defaultMessageStore.getStoreCheckpoint().setPhysicMsgTimestamp(storeTimestamp);
                    }

                    this.requestsRead.clear();
                } else {
                    // Because of individual messages is set to not sync flush, it
                    // will come to this process
                    CommitLog.this.mappedFileQueue.flush(0);
                }
            }
        }

        public void run() {
            CommitLog.log.info(this.getServiceName() + " service started");

            while (!this.isStopped()) {
                try {
                    this.waitForRunning(10);
                    this.doCommit();
                } catch (Exception e) {
                    CommitLog.log.warn(this.getServiceName() + " service has exception. ", e);
                }
            }

            // Under normal circumstances shutdown, wait for the arrival of the
            // request, and then flush
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                CommitLog.log.warn("GroupCommitService Exception, ", e);
            }

            synchronized (this) {
                this.swapRequests();
            }

            this.doCommit();

            CommitLog.log.info(this.getServiceName() + " service end");
        }

        @Override
        protected void onWaitEnd() {
            this.swapRequests();
        }

        @Override
        public String getServiceName() {
            return GroupCommitService.class.getSimpleName();
        }

        @Override
        public long getJointime() {
            return 1000 * 60 * 5;
        }
    }  //end GroupCommitService    
    
    
    
    // ***************************AppendMessageCallback ************************
   
    class DefaultAppendMessageCallback implements AppendMessageCallback {
        // File at the end of the minimum fixed length empty
        private static final int END_FILE_MIN_BLANK_LENGTH = 4 + 4;
        private final ByteBuffer msgIdMemory;
        // Store the message content
        private final ByteBuffer msgStoreItemMemory;
        // The maximum length of the message
        private final int maxMessageSize;
        // Build Message Key
        private final StringBuilder keyBuilder = new StringBuilder();

        private final StringBuilder msgIdBuilder = new StringBuilder();

        private final ByteBuffer hostHolder = ByteBuffer.allocate(8);

        DefaultAppendMessageCallback(final int size) {
            this.msgIdMemory = ByteBuffer.allocate(MessageDecoder.MSG_ID_LENGTH);
            this.msgStoreItemMemory = ByteBuffer.allocate(size + END_FILE_MIN_BLANK_LENGTH);
            this.maxMessageSize = size;
        }

        public ByteBuffer getMsgStoreItemMemory() {
            return msgStoreItemMemory;
        }

        // 18.4 默认的追回消息回调
        public AppendMessageResult doAppend(
            // 0
            final long fileFromOffset, 
            // 临时申请的缓冲区(position == 上次写入的字节数),此处为:0
            final ByteBuffer byteBuffer, 
            // 10485760(文件最大的位置)
            final int maxBlank,
            // 具体的消息内容
            final MessageExtBrokerInner msgInner) {
            // STORETIMESTAMP + STOREHOSTADDRESS + OFFSET <br>

            // PHY OFFSET
            // wroteOffset  = 0
            long wroteOffset = fileFromOffset + byteBuffer.position();
            // 18.5 
            this.resetByteBuffer(hostHolder, 8);
            
            // msgId  = ip+port+wroteOffset
            // 0A00065200001FBB0000000000000000
            String msgId = MessageDecoder.createMessageId(
                this.msgIdMemory, 
                // 18.5 
                msgInner.getStoreHostBytes(hostHolder), 
                wroteOffset );
            
            
            // FooBar-0
            // Record ConsumeQueue information
            keyBuilder.setLength(0);
            keyBuilder.append(msgInner.getTopic());
            keyBuilder.append('-');
            keyBuilder.append(msgInner.getQueueId());
            // Topic + "-" + QueueId
            // 主题名称+队列id
            String key = keyBuilder.toString();
            // 找到主题对应的offset
            Long queueOffset = CommitLog.this.topicQueueTable.get(key);
            // 主题对应的queueOffset不存在的情况下
            if (null == queueOffset) {
                // { FooBar-0 = 0 }
                queueOffset = 0L;
                CommitLog.this.topicQueueTable.put(key, queueOffset);
            }
            
            // 获取消息的类型
            // Transaction messages that require special handling
            final int tranType = MessageSysFlag.getTransactionValue(msgInner.getSysFlag());
            switch (tranType) {
                // Prepared and Rollback message is not consumed, will not enter the
                // consumer queuec
                case MessageSysFlag.TRANSACTION_PREPARED_TYPE:
                case MessageSysFlag.TRANSACTION_ROLLBACK_TYPE:
                    queueOffset = 0L;
                    break;
                case MessageSysFlag.TRANSACTION_NOT_TYPE:
                case MessageSysFlag.TRANSACTION_COMMIT_TYPE:
                default:
                    break;
            }

            /**
             * Serialize message
             */
            // 获取消息的properties
            // propertiesData  == nul;
            final byte[] propertiesData =
                msgInner.getPropertiesString() == null ? null : msgInner.getPropertiesString().getBytes(MessageDecoder.CHARSET_UTF8);
            // propertiesLength  == 0
            final int propertiesLength = propertiesData == null ? 0 : propertiesData.length;
            // 属性字节数不能 > 32767
            // 0 > 32767
            if (propertiesLength > Short.MAX_VALUE) {
                log.warn("putMessage message properties length too long. length={}", propertiesData.length);
                return new AppendMessageResult(AppendMessageStatus.PROPERTIES_SIZE_EXCEEDED);
            }
            
            // Topic信息
            // FooBar
            // [70, 111, 111, 66, 97, 114]
            final byte[] topicData = msgInner.getTopic().getBytes(MessageDecoder.CHARSET_UTF8);
            // topicLength  == 6
            final int topicLength = topicData.length;

            // bodyLength  == 32
            final int bodyLength = msgInner.getBody() == null ? 0 : msgInner.getBody().length;
            
            // ************************计算出消息的总长度********************************
            // 18.6  调用消息长度.
            // msgLen(129)  == (32,6,0)
            final int msgLen = calMsgLength(
                // bodyLength = 32
               bodyLength, 
               // topicLength = 6
               topicLength, 
               // propertiesLength = 0
               propertiesLength);

            // Exceeds the maximum message
            // 18.7 对消息的长度进行限制(消息长度不能 > 4m).
            // maxMessageSize = 4194304
            if (msgLen > this.maxMessageSize) {
                CommitLog.log.warn("message size exceeded, msg total size: " + msgLen + ", msg body size: " + bodyLength
                    + ", maxMessageSize: " + this.maxMessageSize);
                return new AppendMessageResult(AppendMessageStatus.MESSAGE_SIZE_EXCEEDED);
            } 
            
            // 生产者一次提交的消息,判断这条消息:是否有足够的磁盘空间
            // Determines whether there is sufficient free space
            if ((msgLen + END_FILE_MIN_BLANK_LENGTH) > maxBlank) {
                this.resetByteBuffer(this.msgStoreItemMemory, maxBlank);
                // 1 TOTALSIZE
                this.msgStoreItemMemory.putInt(maxBlank);
                // 2 MAGICCODE
                this.msgStoreItemMemory.putInt(CommitLog.BLANK_MAGIC_CODE);
                // 3 The remaining space may be any value
                // Here the length of the specially set maxBlank
                final long beginTimeMills = CommitLog.this.defaultMessageStore.now();
                byteBuffer.put(this.msgStoreItemMemory.array(), 0, maxBlank);
                return new AppendMessageResult(AppendMessageStatus.END_OF_FILE, wroteOffset, maxBlank, msgId, msgInner.getStoreTimestamp(),
                    queueOffset, CommitLog.this.defaultMessageStore.now() - beginTimeMills);
            }
            
            // Initialization of storage space
            this.resetByteBuffer(msgStoreItemMemory, msgLen);
            // 1 TOTALSIZE = 129
            this.msgStoreItemMemory.putInt(msgLen);
            // 固定的一个Magiccode
            // 2 MAGICCODE = -626843481
            this.msgStoreItemMemory.putInt(CommitLog.MESSAGE_MAGIC_CODE);
            // 3 BODYCRC = 151131488
            this.msgStoreItemMemory.putInt(msgInner.getBodyCRC());
            // 4 QUEUEID[消息队列的ID] = 0
            this.msgStoreItemMemory.putInt(msgInner.getQueueId());
            // 5 FLAG = 0
            this.msgStoreItemMemory.putInt(msgInner.getFlag());
            
            // FooBar-0 = 0
            // 队列的消息偏移量(注意:队列名称 = 主题名称-队列下标)
            // 6 QUEUEOFFSET = 0
            this.msgStoreItemMemory.putLong(queueOffset);
            
            // commitLog的起始位置
            // 7 PHYSICALOFFSET = 0
            this.msgStoreItemMemory.putLong(fileFromOffset + byteBuffer.position());
            
            // 8 SYSFLAG = 0
            this.msgStoreItemMemory.putInt(msgInner.getSysFlag());
            
            // 9 BORNTIMESTAMP = 1575884746075
            this.msgStoreItemMemory.putLong(msgInner.getBornTimestamp());
            // 10 BORNHOST = [127, 0, 0, 1, 0, 0, 0, 0]
            this.resetByteBuffer(hostHolder, 8);
            this.msgStoreItemMemory.put(msgInner.getBornHostBytes(hostHolder));
            
            // 11 STORETIMESTAMP = 1575884746086
            this.msgStoreItemMemory.putLong(msgInner.getStoreTimestamp());
            
            // 12 STOREHOSTADDRESS = [10, 0, 6, 82, 0, 0, 31, -69]
            this.resetByteBuffer(hostHolder, 8);
            this.msgStoreItemMemory.put(msgInner.getStoreHostBytes(hostHolder));
            //this.msgBatchMemory.put(msgInner.getStoreHostBytes());
           
             // 13 RECONSUMETIMES = 0
            this.msgStoreItemMemory.putInt(msgInner.getReconsumeTimes());
            // 14 Prepared Transaction Offset = 0
            this.msgStoreItemMemory.putLong(msgInner.getPreparedTransactionOffset());
            // 15 BODY = 32
            this.msgStoreItemMemory.putInt(bodyLength);
            if (bodyLength > 0)
                this.msgStoreItemMemory.put(msgInner.getBody());
                
            // 16 TOPIC = 6
            this.msgStoreItemMemory.put((byte) topicLength);
            this.msgStoreItemMemory.put(topicData);
            
            // 17 PROPERTIES = 0
            this.msgStoreItemMemory.putShort((short) propertiesLength);
            if (propertiesLength > 0)
                this.msgStoreItemMemory.put(propertiesData);

            final long beginTimeMills = CommitLog.this.defaultMessageStore.now();
            // Write messages to the queue buffer
            byteBuffer.put(this.msgStoreItemMemory.array(), 0, msgLen);

            // 18. 创建追回消息的返回结果
            AppendMessageResult result = new AppendMessageResult(
                   // PUT_OK
                   AppendMessageStatus.PUT_OK, 
                   // 0
                   wroteOffset, 
                   // 129
                   msgLen, 
                   // 0A00065200001FBB0000000000000000
                   msgId,
                   // 1575884746086
                   msgInner.getStoreTimestamp(), 
                   // 0
                   queueOffset, 
                   // 310899
                   CommitLog.this.defaultMessageStore.now() - beginTimeMills
           );

            switch (tranType) {
                case MessageSysFlag.TRANSACTION_PREPARED_TYPE:
                case MessageSysFlag.TRANSACTION_ROLLBACK_TYPE:
                    break;
                case MessageSysFlag.TRANSACTION_NOT_TYPE:
                case MessageSysFlag.TRANSACTION_COMMIT_TYPE:
                    // 19.更新队列的偏移量(自增)
                    // The next update ConsumeQueue information
                    // 更新ConsumeQueue队列的偏移量
                    // FooBar-0 = 1
                    CommitLog.this.topicQueueTable.put(key, ++queueOffset);
                    break;
                default:
                    break;
            }
            return result;
        }

        public AppendMessageResult doAppend(final long fileFromOffset, final ByteBuffer byteBuffer, final int maxBlank,
            final MessageExtBatch messageExtBatch) {
            byteBuffer.mark();
            //physical offset
            long wroteOffset = fileFromOffset + byteBuffer.position();
            // Record ConsumeQueue information
            keyBuilder.setLength(0);
            keyBuilder.append(messageExtBatch.getTopic());
            keyBuilder.append('-');
            keyBuilder.append(messageExtBatch.getQueueId());
            String key = keyBuilder.toString();
            Long queueOffset = CommitLog.this.topicQueueTable.get(key);
            if (null == queueOffset) {
                queueOffset = 0L;
                CommitLog.this.topicQueueTable.put(key, queueOffset);
            }
            long beginQueueOffset = queueOffset;
            int totalMsgLen = 0;
            int msgNum = 0;
            msgIdBuilder.setLength(0);
            final long beginTimeMills = CommitLog.this.defaultMessageStore.now();
            ByteBuffer messagesByteBuff = messageExtBatch.getEncodedBuff();
            this.resetByteBuffer(hostHolder, 8);
            ByteBuffer storeHostBytes = messageExtBatch.getStoreHostBytes(hostHolder);
            messagesByteBuff.mark();
            while (messagesByteBuff.hasRemaining()) {
                // 1 TOTALSIZE
                final int msgPos = messagesByteBuff.position();
                final int msgLen = messagesByteBuff.getInt();
                final int bodyLen = msgLen - 40; //only for log, just estimate it
                // Exceeds the maximum message
                if (msgLen > this.maxMessageSize) {
                    CommitLog.log.warn("message size exceeded, msg total size: " + msgLen + ", msg body size: " + bodyLen
                        + ", maxMessageSize: " + this.maxMessageSize);
                    return new AppendMessageResult(AppendMessageStatus.MESSAGE_SIZE_EXCEEDED);
                }
                totalMsgLen += msgLen;
                // Determines whether there is sufficient free space
                if ((totalMsgLen + END_FILE_MIN_BLANK_LENGTH) > maxBlank) {
                    this.resetByteBuffer(this.msgStoreItemMemory, 8);
                    // 1 TOTALSIZE
                    this.msgStoreItemMemory.putInt(maxBlank);
                    // 2 MAGICCODE
                    this.msgStoreItemMemory.putInt(CommitLog.BLANK_MAGIC_CODE);
                    // 3 The remaining space may be any value
                    //ignore previous read
                    messagesByteBuff.reset();
                    // Here the length of the specially set maxBlank
                    byteBuffer.reset(); //ignore the previous appended messages
                    byteBuffer.put(this.msgStoreItemMemory.array(), 0, 8);
                    return new AppendMessageResult(AppendMessageStatus.END_OF_FILE, wroteOffset, maxBlank, msgIdBuilder.toString(), messageExtBatch.getStoreTimestamp(),
                        beginQueueOffset, CommitLog.this.defaultMessageStore.now() - beginTimeMills);
                }
                //move to add queue offset and commitlog offset
                messagesByteBuff.position(msgPos + 20);
                messagesByteBuff.putLong(queueOffset);
                messagesByteBuff.putLong(wroteOffset + totalMsgLen - msgLen);

                storeHostBytes.rewind();
                String msgId = MessageDecoder.createMessageId(this.msgIdMemory, storeHostBytes, wroteOffset + totalMsgLen - msgLen);
                if (msgIdBuilder.length() > 0) {
                    msgIdBuilder.append(',').append(msgId);
                } else {
                    msgIdBuilder.append(msgId);
                }
                queueOffset++;
                msgNum++;
                messagesByteBuff.position(msgPos + msgLen);
            }

            messagesByteBuff.position(0);
            messagesByteBuff.limit(totalMsgLen);
            byteBuffer.put(messagesByteBuff);
            messageExtBatch.setEncodedBuff(null);
            AppendMessageResult result = new AppendMessageResult(AppendMessageStatus.PUT_OK, wroteOffset, totalMsgLen, msgIdBuilder.toString(),
                messageExtBatch.getStoreTimestamp(), beginQueueOffset, CommitLog.this.defaultMessageStore.now() - beginTimeMills);
            result.setMsgNum(msgNum);
            CommitLog.this.topicQueueTable.put(key, queueOffset);

            return result;
        }

        private void resetByteBuffer(final ByteBuffer byteBuffer, final int limit) {
            byteBuffer.flip();
            byteBuffer.limit(limit);
        }

    } //end AppendMessageCallback
    
    
    // ************************************计算消息的长度*******************************
    // 18.6.1 计算消息的总长度
    private static int calMsgLength(
         // bodyLength = 32
         int bodyLength, 
         // topicLength = 6
         int topicLength, 
         // propertiesLength = 0
         int propertiesLength) {
             
        // TOTALSIZE       = 整个消息的长度(用于界定消息的边界)[4字节]
        // MAGICCODE       = 魔数[4字节]
        // BODYCRC          = 消息体crc32校验码[4字节]
        // QUEUEID          = 消息消费队列ID[4字节]
        // FLAG             = RocketMQ不做处理,预留给应用程序使用[4字节]
        // QUEUEOFFSET      = 消息在消息队列的偏移量[8字节]
        // PHYSICALOFFSET   = 消息在CommitLog的偏移量[8字节]
        // SYSFLAG          = 消息系统Flag(是否压缩/是否支持事务)[4字节]
        // BORNTIMESTAMP    = 生产者调用消息发送API的时间[8字节]
        // BORNHOST         = 生产者IP和端口
        // STORETIMESTAMP   = 消息存储时间[8字节]
        // STOREHOSTADDRESS = Brokder服务器的IP+端口[8字节]
        // RECONSUMETIMES   = 消息重试次数[4字节]
        // Prepared Transaction Offset = 事务消息物理偏移量[8字节]
        // BODY             = 消息体内容的长度[4字节]
        // TOPIC            = 消息主题的长度[1字节] 
        // propertiesLength = 消息属性长度[2字节],消息属性长度不能超过65536个字符
        //       
             
        final int msgLen = 4 //TOTALSIZE
            + 4 //MAGICCODE
            + 4 //BODYCRC
            + 4 //QUEUEID
            + 4 //FLAG
            + 8 //QUEUEOFFSET
            + 8 //PHYSICALOFFSET
            + 4 //SYSFLAG
            + 8 //BORNTIMESTAMP
            + 8 //BORNHOST
            + 8 //STORETIMESTAMP
            + 8 //STOREHOSTADDRESS
            + 4 //RECONSUMETIMES
            + 8 //Prepared Transaction Offset
            + 4 + (bodyLength > 0 ? bodyLength : 0) //BODY
            + 1 + topicLength //TOPIC
            + 2 + (propertiesLength > 0 ? propertiesLength : 0) //propertiesLength
            + 0;
        return msgLen;
    }
    
}    
```
### (10). MappedFileQueue
```
package org.apache.rocketmq.store;
public class MappedFileQueue {
    
    // commit文件列表
    private final CopyOnWriteArrayList<MappedFile> mappedFiles = new CopyOnWriteArrayList<MappedFile>();
    
    // 构造器
     public MappedFileQueue(
         // C:\Users\lixin\store\commitlog
         final String storePath, 
         // 10485760
         int mappedFileSize,
         // org.apache.rocketmq.store.AllocateMappedFileService
         // 分配MappedFile管理
         AllocateMappedFileService allocateMappedFileService) {
          this.storePath = storePath;
          this.mappedFileSize = mappedFileSize;
          this.allocateMappedFileService = allocateMappedFileService;
    }
    
    // 12. 获取最后一个MappedFile文件
     public MappedFile getLastMappedFile(final long startOffset) {
        // 12.1 startOffset = 0
        return getLastMappedFile(startOffset, true);
    }
    
    // 12.2 
    public MappedFile getLastMappedFile(
         // 0
         final long startOffset, 
         // true
         boolean needCreate) {
             
        long createOffset = -1;
        
        // 12.3 获取最后一个MappedFile
        // null
        MappedFile mappedFileLast = getLastMappedFile();
        
        // 如果获取MappedFile,代表是第一次请求
        if (mappedFileLast == null) {
            // 0 = 0 - 0;
            createOffset = startOffset - (startOffset % this.mappedFileSize);
        }
        
        // 如果最后一个文件存在,则计算出下一个文件的起始offset
        if (mappedFileLast != null && mappedFileLast.isFull()) {
            // 假设:mappedFileLast.getFileFromOffset() = 00000000000010485760
            // 假设:mappedFileSize = 10485760
            // createOffset = 20971520
            createOffset = mappedFileLast.getFileFromOffset() + this.mappedFileSize;
        }
        
        // 
        if (createOffset != -1 && needCreate) {
            // C:\Users\lixin\store\commitlog\00000000000000000000
            String nextFilePath = this.storePath + File.separator + UtilAll.offset2FileName(createOffset);
            // C:\Users\lixin\store\commitlog\00000000000010485760
            String nextNextFilePath = this.storePath + File.separator
                + UtilAll.offset2FileName(createOffset + this.mappedFileSize);
                
            MappedFile mappedFile = null;
            // 有指定MappedFile分配服务
            if (this.allocateMappedFileService != null) {  
                //******************************************************
                // 13. 委托给:AllocateMappedFileService创建文件
                // 交给MappedFile创建MappedFile
                mappedFile = this.allocateMappedFileService
                    .putRequestAndReturnMappedFile(
                        // C:\Users\lixin\store\commitlog\00000000000000000000
                        nextFilePath,
                        // C:\Users\lixin\store\commitlog\00000000000010485760
                        nextNextFilePath, 
                        // 10485760(1G)
                        this.mappedFileSize);
            } else {
                try {
                    mappedFile = new MappedFile(nextFilePath, this.mappedFileSize);
                } catch (IOException e) {
                    log.error("create mappedFile exception", e);
                }
            }
            
            // 17. 异步创建任务成功
            if (mappedFile != null) {  //true
                if (this.mappedFiles.isEmpty()) {  // true
                    mappedFile.setFirstCreateInQueue(true);
                } 
                // 17.1 添加到映射文件集合
                this.mappedFiles.add(mappedFile);
            }            
            return mappedFile;
        }
        return mappedFileLast;
    }
    
    // 12.3 获取最后一个commit文件
    public MappedFile getLastMappedFile() {
        MappedFile mappedFileLast = null;
        // 判断mappedFiles集合不为空则获取集合中最后一个文件
        while (!this.mappedFiles.isEmpty()) { //false
            try {
                mappedFileLast = this.mappedFiles.get(this.mappedFiles.size() - 1);
                break;
            } catch (IndexOutOfBoundsException e) {
                //continue;
            } catch (Exception e) {
                log.error("getLastMappedFile has exception.", e);
                break;
            }
        }
        return mappedFileLast;
    }
    
}
```
### (11). AllocateMappedFileService
```
package org.apache.rocketmq.store;

// ServiceThread implements Runnable
public class AllocateMappedFileService extends ServiceThread {
     private static int waitTimeOut = 1000 * 5;
     
     private ConcurrentMap<String, AllocateRequest> requestTable 
       = new ConcurrentHashMap<String, AllocateRequest>();
       
    private PriorityBlockingQueue<AllocateRequest> requestQueue 
       = new PriorityBlockingQueue<AllocateRequest>();
    
    private volatile boolean hasException = false;
    private DefaultMessageStore messageStore;
    
    // DefaultMessageStore 
    public AllocateMappedFileService(DefaultMessageStore messageStore) {
        this.messageStore = messageStore;
    }
    
    // 14. 添加请求并且返回:MappedFile
    public MappedFile putRequestAndReturnMappedFile(
        // C:\Users\lixin\store\commitlog\00000000000000000000
        String nextFilePath, 
        // C:\Users\lixin\store\commitlog\00000000000010485760
        String nextNextFilePath, 
        // 10485760
        int fileSize) {
        
        int canSubmitRequests = 2;
        // false
        if (this.messageStore.getMessageStoreConfig().isTransientStorePoolEnable()) {
            if (this.messageStore.getMessageStoreConfig().isFastFailIfNoBufferInStorePool()
                && BrokerRole.SLAVE != this.messageStore.getMessageStoreConfig().getBrokerRole()) { //if broker is slave, don't fast fail even no buffer in pool
                canSubmitRequests = this.messageStore.getTransientStorePool().remainBufferNumbs() - this.requestQueue.size();
            }
        }
        
        // 14.1  创建分配请求
        // nextFilePath = C:\Users\lixin\store\commitlog\00000000000000000000
        // fileSize = 10485760
        AllocateRequest nextReq = new AllocateRequest(nextFilePath, fileSize);
        // 如果map中不存在:nextFilePath这个key,则put进去,并返回:true
        // 如果map中存在:nextFilePath这个key,则返回:false
        boolean nextPutOK = this.requestTable.putIfAbsent(nextFilePath, nextReq) == null;
        
        // true
        if (nextPutOK) {
            if (canSubmitRequests <= 0) {
                log.warn("[NOTIFYME]TransientStorePool is not enough, so create mapped file error, " +
                    "RequestQueueSize : {}, StorePoolSize: {}", this.requestQueue.size(), this.messageStore.getTransientStorePool().remainBufferNumbs());
                this.requestTable.remove(nextFilePath);
                return null;
            }
            // 往队列里进行添加
            boolean offerOK = this.requestQueue.offer(nextReq);
            if (!offerOK) {
                log.warn("never expected here, add a request to preallocate queue failed");
            }
            canSubmitRequests--;
        }
        
        // C:\Users\lixin\store\commitlog\00000000000010485760
        AllocateRequest nextNextReq = new AllocateRequest(nextNextFilePath, fileSize);
        boolean nextNextPutOK = this.requestTable.putIfAbsent(nextNextFilePath, nextNextReq) == null;
        // true
        if (nextNextPutOK) {
            if (canSubmitRequests <= 0) {
                log.warn("[NOTIFYME]TransientStorePool is not enough, so skip preallocate mapped file, " +
                    "RequestQueueSize : {}, StorePoolSize: {}", this.requestQueue.size(), this.messageStore.getTransientStorePool().remainBufferNumbs());
                this.requestTable.remove(nextNextFilePath);
            } else {
                boolean offerOK = this.requestQueue.offer(nextNextReq);
                if (!offerOK) {
                    log.warn("never expected here, add a request to preallocate queue failed");
                }
            }
        }
        
         if (hasException) {
             // 如果有错误,抛出异常
            log.warn(this.getServiceName() + " service has exception. so return null");
            return null;
        }
        
        // 14.2 获取结果:C:\Users\lixin\store\commitlog\00000000000000000000
        AllocateRequest result = this.requestTable.get(nextFilePath);
        try {
            if (result != null) {
                // 16. 获得CountdownLatch,并等待watiTimeOut毫秒
                boolean waitOK = result.getCountDownLatch()
                                 .await(waitTimeOut, TimeUnit.MILLISECONDS);
                if (!waitOK) {  // true
                    // 提示创建mmap文件超时
                    log.warn("create mmap timeout " + result.getFilePath() + " " + result.getFileSize());
                    return null;
                } else {
                    //  从map中移除
                    //  16.1 并返回:MappedFile
                    this.requestTable.remove(nextFilePath);
                    return result.getMappedFile();
                }
            } else {
                log.error("find preallocate mmap failed, this never happen");
            }
        } catch (InterruptedException e) {
            log.warn(this.getServiceName() + " service has exception. ", e);
        }

        return null;
        
          
    } //end putRequestAndReturnMappedFile
    
    
    public void run() {
        log.info(this.getServiceName() + " service started");
        while (  !this.isStopped() 
                  // 15. 将文件映射进内存
                 && this.mmapOperation()) {
        }
        log.info(this.getServiceName() + " service end");
    }    
    
    // 15.1 将文件映射进内存
    private boolean mmapOperation() {
        boolean isSuccess = false;
        AllocateRequest req = null;
        try {
            // 等待从队列中弹出数据(阻塞线程往下执行)
            req = this.requestQueue.take();
            // 分配请求
            AllocateRequest expectedRequest = this.requestTable.get(req.getFilePath());
            if (null == expectedRequest) {
                log.warn("this mmap request expired, maybe cause timeout " + req.getFilePath() + " "
                    + req.getFileSize());
                return true;
            }
            if (expectedRequest != req) {
                log.warn("never expected here,  maybe cause timeout " + req.getFilePath() + " "
                    + req.getFileSize() + ", req:" + req + ", expectedRequest:" + expectedRequest);
                return true;
            }

            // true
            if (req.getMappedFile() == null) {                
                long beginTime = System.currentTimeMillis();

                MappedFile mappedFile;
                // false
                if (messageStore.getMessageStoreConfig().isTransientStorePoolEnable()) {
                    try {
                        mappedFile = ServiceLoader.load(MappedFile.class).iterator().next();
                        mappedFile.init(req.getFilePath(), req.getFileSize(), messageStore.getTransientStorePool());
                    } catch (RuntimeException e) {
                        log.warn("Use default implementation.");
                        mappedFile = new MappedFile(req.getFilePath(), req.getFileSize(), messageStore.getTransientStorePool());
                    }
                } else {
                    // 15.2 创建:MappedFile
                    mappedFile = new MappedFile(req.getFilePath(), req.getFileSize());
                }

               
                long eclipseTime = UtilAll.computeEclipseTimeMilliseconds(beginTime);
                if (eclipseTime > 10) {
                    int queueSize = this.requestQueue.size();
                    log.warn("create mappedFile spent time(ms) " + eclipseTime + " queue size " + queueSize
                        + " " + req.getFilePath() + " " + req.getFileSize());
                }

                // 15.4 
                // mappedFile.getFileSize() == 10485760
                // this.messageStore.getMessageStoreConfig().getMapedFileSizeCommitLog() == 10485760
                // false
                // messageStore.getMessageStoreConfig().isWarmMapedFileEnable() == 15.4 
                if (mappedFile.getFileSize() >= this.messageStore.getMessageStoreConfig()
                    .getMapedFileSizeCommitLog()
                    &&
                    this.messageStore.getMessageStoreConfig().isWarmMapedFileEnable()) {
                    mappedFile.warmMappedFile(this.messageStore.getMessageStoreConfig().getFlushDiskType(),
                        this.messageStore.getMessageStoreConfig().getFlushLeastPagesWhenWarmMapedFile());
                }
                // 设置 AllocateRequest  对象的MappedFile
                req.setMappedFile(mappedFile);
                this.hasException = false;
                isSuccess = true;
            }
        } catch (InterruptedException e) {
            log.warn(this.getServiceName() + " interrupted, possibly by shutdown.");
            // 设置有异常
            this.hasException = true;
            return false;
        } catch (IOException e) {
            log.warn(this.getServiceName() + " service has exception. ", e);
              // 设置有异常
            this.hasException = true;
            if (null != req) {
                requestQueue.offer(req);
                try {
                    Thread.sleep(1);
                } catch (InterruptedException ignored) {
                }
            }
        } finally {
            if (req != null && isSuccess)
                // 15.5 设置请求完成
                req.getCountDownLatch().countDown();
        }
        return true;
    }
    
    
    static class AllocateRequest implements Comparable<AllocateRequest> {
        // Full file path
        private String filePath;
        private int fileSize;
        private CountDownLatch countDownLatch = new CountDownLatch(1);
        private volatile MappedFile mappedFile = null;

        public AllocateRequest(String filePath, int fileSize) {
            this.filePath = filePath;
            this.fileSize = fileSize;
        }

        public String getFilePath() {
            return filePath;
        }

        public void setFilePath(String filePath) {
            this.filePath = filePath;
        }

        public int getFileSize() {
            return fileSize;
        }

        public void setFileSize(int fileSize) {
            this.fileSize = fileSize;
        }

        public CountDownLatch getCountDownLatch() {
            return countDownLatch;
        }

        public void setCountDownLatch(CountDownLatch countDownLatch) {
            this.countDownLatch = countDownLatch;
        }

        public MappedFile getMappedFile() {
            return mappedFile;
        }

        public void setMappedFile(MappedFile mappedFile) {
            this.mappedFile = mappedFile;
        }
                
    } //end AllocateRequest
    
}
```
### (12). MappedFile
```
package org.apache.rocketmq.store;

public class MappedFile extends ReferenceResource {
    // 总的虚拟内存大小
    private static final AtomicLong TOTAL_MAPPED_VIRTUAL_MEMORY = new AtomicLong(0);
    // 文件个数
    private static final AtomicInteger TOTAL_MAPPED_FILES = new AtomicInteger(0);
    
    // 
   protected final AtomicInteger wrotePosition = new AtomicInteger(0);
   protected final AtomicInteger committedPosition = new AtomicInteger(0);
   private final AtomicInteger flushedPosition = new AtomicInteger(0);
    
    // 文件名称
   private String fileName;
   // 文件的起始位置
   private long fileFromOffset;
   // 文件
   private File file;
   // 文件大小
   protected int fileSize;
    
   private MappedByteBuffer mappedByteBuffer;
    
   protected FileChannel fileChannel;
    
    
    // 15.2 创建MappedFile
    public MappedFile(final String fileName, final int fileSize) throws IOException {
        init(fileName, fileSize);
    }
    
    // 15.3 初始化方法
    private void init(final String fileName, final int fileSize) throws IOException {
        this.fileName = fileName;
        this.fileSize = fileSize;
        // C:\Users\lixin\store\commitlog\00000000000000000000
        this.file = new File(fileName);
        // fileFromOffset(0)  = Long.parseLong("00000000000000000000")
        this.fileFromOffset = Long.parseLong(this.file.getName());
        boolean ok = false;
        // 递归创建目录:C:\Users\lixin\store\commitlog
        ensureDirOK(this.file.getParent());

        try {
            // 创建文件channel
            this.fileChannel = new RandomAccessFile(this.file, "rw").getChannel();
            // RandomAccessFile获得的Channel可以开启任意三种模式(MapMode)
            // InputStream获得的Channel只能开启:MapMode.READ_ONLY
            // 将fileChannel通道与内存进行映射
            // MapMode.READ_WRITE :  可读写操作
            // position(0) :         文件的起始位置
            // size(10485760) :      size字节
            this.mappedByteBuffer = this.fileChannel.map(MapMode.READ_WRITE, 0, fileSize);
            
            // fileSize = 10485760
            // 设置总的内存
            TOTAL_MAPPED_VIRTUAL_MEMORY.addAndGet(fileSize);
            // 总的Mapped文件数
            TOTAL_MAPPED_FILES.incrementAndGet();
            ok = true;
        } catch (FileNotFoundException e) {
            log.error("create file channel " + this.fileName + " Failed. ", e);
            throw e;
        } catch (IOException e) {
            log.error("map file " + this.fileName + " Failed. ", e);
            throw e;
        } finally {
            // 如果不成功的情况下,关闭流
            if (!ok && this.fileChannel != null) {
                this.fileChannel.close();
            }
        }
    } //end init
    
    // 18.1 追加消息
    public AppendMessageResult appendMessage(final MessageExtBrokerInner msg, final AppendMessageCallback cb) {
        return appendMessagesInner(msg, cb);
    }  //end appendMessage
    
    // 18.2 追加消息
    public AppendMessageResult appendMessagesInner(final MessageExt messageExt, final AppendMessageCallback cb) {
        assert messageExt != null;
        assert cb != null;
        
        // 18.3 获取当前的位置
        // currentPos == 0
        int currentPos = this.wrotePosition.get();

        // 0 < 10485760
        if (currentPos < this.fileSize) {
            // writeBuffer == null
            // mappedByteBuffer.slice()返回原ByteBuffer的一个镜像,所有改变互相可见.position和limit独立.
            ByteBuffer byteBuffer = writeBuffer != null ? writeBuffer.slice() : this.mappedByteBuffer.slice();
            // 设置position = 0
            byteBuffer.position(currentPos);
            
            // 定义结果集
            AppendMessageResult result = null;
            if (messageExt instanceof MessageExtBrokerInner) {  // 内联消息
                // *****************追加消息*******************************
                // 18.4 追回消息
                result = cb.doAppend(this.getFileFromOffset(), byteBuffer, this.fileSize - currentPos, (MessageExtBrokerInner) messageExt);
            } else if (messageExt instanceof MessageExtBatch) {   // 批量消息
                result = cb.doAppend(this.getFileFromOffset(), byteBuffer, this.fileSize - currentPos, (MessageExtBatch) messageExt);
            } else {
                return new AppendMessageResult(AppendMessageStatus.UNKNOWN_ERROR);
            }
            // *******************重点************************            
            // 20 记录消息上次写的位置,下次从这个位置开始追加消息
            // DefaultMessageStore.FlushConsumeQueueService线程每隔一秒
            // 实现对consumeQueue/Index进行刷盘.
            this.wrotePosition.addAndGet(result.getWroteBytes());
            // 获得写入的时间
            this.storeTimestamp = result.getStoreTimestamp();
            return result;
        }
        log.error("MappedFile.appendMessage return null, wrotePosition: {} fileSize: {}", currentPos, this.fileSize);
        return new AppendMessageResult(AppendMessageStatus.UNKNOWN_ERROR);
    } //end appendMessagesInner
    
}
```
### (13). MessageExt
```
public class MessageExt extends Message {
    
    // 18.5 获得IP地址写入到:byteBuffer
    public ByteBuffer getStoreHostBytes(ByteBuffer byteBuffer) {
        return socketAddress2ByteBuffer(this.storeHost, byteBuffer);
    }
    
    // 18.5.1 获得IP地址写入到:byteBuffer
    public static ByteBuffer socketAddress2ByteBuffer(final SocketAddress socketAddress, final ByteBuffer byteBuffer) {
        InetSocketAddress inetSocketAddress = (InetSocketAddress) socketAddress;
        // [10],[0],[6],[82]
        byteBuffer.put(inetSocketAddress.getAddress().getAddress(), 0, 4);
        // int 4字节
        // [8123]
        byteBuffer.putInt(inetSocketAddress.getPort());
        byteBuffer.flip();
        return byteBuffer;
    }
    
}
```
### (14). MessageDecoder
```
public class MessageDecoder {
    
    // 
    public static String createMessageId(
        // 
        final ByteBuffer input, 
        final ByteBuffer addr, 
        final long offset) {
            
        input.flip();input.limit(MessageDecoder.MSG_ID_LENGTH);
        // [10, 0, 6, 82, 0, 0, 31, -69]
        input.put(addr);
        // long 8字节
        // offset = 0
        // [10, 0, 6, 82, 0, 0, 31, -69, 0, 0, 0, 0, 0, 0, 0, 0]
        input.putLong(offset);
        // 0A00065200001FBB0000000000000000
        return UtilAll.bytes2string(input.array());
    }
}
```
### (15). MappedFile 写文件,同时记录:wrotePosition = 上次写的位置

```
public class MappedFile extends ReferenceResource {
    
    public AppendMessageResult appendMessagesInner(final MessageExt messageExt, final AppendMessageCallback cb) {
        assert messageExt != null;
        assert cb != null;
        int currentPos = this.wrotePosition.get();
        
        if (currentPos < this.fileSize) {
             // ......
            // *****************************重点****************
            this.wrotePosition.addAndGet(result.getWroteBytes());
           // *****************************重点****************
           // ......
        }
       // ......
    } 
}
```
### (16). 定时任务:DefaultMessageStore.ReputMessageService 每隔一秒钟刷盘(写consumeQueue/Index).
```
public class DefaultMessageStore implements MessageStore {
    // ReputMessageService 负责对(ConsumeQueue/Index)进行刷盘
    class ReputMessageService extends ServiceThread {
        
        public void run() {
            while (!this.isStopped()) {
                try {
                    // 休眠一秒再进行业务处理
                    Thread.sleep(1);
                    this.doReput();
                } catch (Exception e) {
                    // ...
                }
            }            
        } //end run
     
      // 判断是否提交日志
      private boolean isCommitLogAvailable() {
            return 
            this.reputFromOffset < 
            // this.writeBuffer == null ? this.wrotePosition.get() : this.committedPosition.get();
            DefaultMessageStore.this.commitLog.getMaxOffset();
      }
        
       private void doReput() {
            for (boolean doNext = true; this.isCommitLogAvailable() && doNext; ) {
                // ...
            }
       } //end doReput
                
  }
}
```
### (17). RoketMQ消息协议( QUEUEOFFSET:为每增加一个条消息,都会进行自增 )
```
     TOTALSIZE       = 整个消息的长度(用于界定消息的边界)[4字节]
     MAGICCODE       = 魔数[4字节]
     BODYCRC          = 消息体crc32校验码[4字节]
     QUEUEID          = 消息消费队列ID[4字节]
     FLAG             = RocketMQ不做处理,预留给应用程序使用[4字节]
     QUEUEOFFSET      = 消息在消息队列的偏移量(consumeQueue)[8字节]
     PHYSICALOFFSET   = 消息在CommitLog的偏移量[8字节]
     SYSFLAG          = 消息系统Flag(是否压缩/是否支持事务)[4字节]
     BORNTIMESTAMP    = 生产者调用消息发送API的时间[8字节]
     BORNHOST         = 生产者IP和端口
     STORETIMESTAMP   = 消息存储时间[8字节]
     STOREHOSTADDRESS = Brokder服务器的IP+端口[8字节]
     RECONSUMETIMES   = 消息重试次数[4字节]
     Prepared Transaction Offset = 事务消息物理偏移量[8字节]
     BODY             = 消息体内容的长度[4字节]
     TOPIC            = 消息主题的长度[1字节] 
     propertiesLength = 消息属性长度[2字节],消息属性长度不能超过65536个字符
```
### (18). ConsumeQueue 数据结构
```
   offset          = 消息在CommitLog的偏移量
   size            = 消息的字节数
   tagsCode        = 消息的标记(*)
```