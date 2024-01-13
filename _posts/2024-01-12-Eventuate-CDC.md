---
layout: post
title: 'Eventuate CDC流程剖析' 
date: 2024-01-12
author: 李新
tags:  Eventuate
---

### (1). 背景

新的项目,对于事务消息还是要求比较严格的,所以,还是需要对Eventuate进行深入的了解和学习.
这一篇主要剖析,Eventuate CDC,是如何订阅binlog和polling消息,并进行发送到MQ(Redis),不涉及底层具体细节,先清楚大概流程. 

### (2). Eventuate CDC源码下载编译打包

```
https://github.com/eventuate-foundation/eventuate-cdc.git

./gradlew  :eventuate-cdc-service:clean :eventuate-cdc-service:assemble
```

### (3). Eventuate CDC 类结构图

![Eventuate 类结构图](/assets/eventuate/imgs/cdc-reader.jpg)

### (4). Eventuate CDC 时序图

![Eventuate时序图](/assets/eventuate/imgs/cdc-reader-sequence.jpg)

### (5). Eventuate CDC 总结
```
1. BinlogEntryReader(PollingDao/MySqlBinaryLogClient/PostgresWalClient),订阅(拉取)消息.  
2. 委派给: BinlogEntryHandler处理.
3. BinlogEntryHandler委派给,CdcDataPublisher进行消息发布
4. CdcDataPublisher最终是委派给:相应的:DataProducer(Redis/ActiveMQ/Kafka/RabbitMQ)处理.

5. Eventuate CDC最终的目的订阅或者拉取消息,并发送到MQ而已.
6. cdc在启动时会创建一个topic(offset.storage.topic ),然后cdc在拉取消息时,每一个channel id(messag表中destination列)都会创建相应的topic. 
7. __consumer_offsets为Kafka自带的offset存储位置.
```

### (6). Eventuate CDC 扩展
如果,想要支持其它MQ(RocketMQ),只需要扩展两个类:DataProducerFactory/DataProducer即可. 
