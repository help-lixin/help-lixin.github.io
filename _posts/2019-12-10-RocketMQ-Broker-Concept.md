---
layout: post
title: 'RocketMQ Broker存储层介绍(一)' 
date: 2019-12-10
author: 李新
tags:  RocketMQ
---

### (1). 介绍
```
RocketMQ主要存储的文件包括:
	commitlog文件
	consumequeue文件
	index文件
```
### (2). commitlog文件
RocketMQ将所有主题的消息存储在同一个文件(commitlog)中,确保消息发送时顺序写入文件,尽最大的能力确保消息发送的高性能和高吞吐量.


```
// commitlog目录结构
+---commitlog
|       00000000000000000000
|       00000000001073741824
```

### (3). consumequeue文件
由于消息中间件一般是基于消息主题的订阅机制,而commitlog是所有消息的集合,这样便给消息按照消息主题检索带来了极大的不便,为了提高消息消费的效率.
RocketMQ引入了:consumequeue文件.每个消息主题(Topic[TopicTest])包含多个消息消费队列(0~n),每一个消息队列有一个消息文件(consumequeue).

```
// consumerqueue 目录结构
+---consumequeue
|   \---TopicTest                              // 消息主题(Topic)
|       \---0                                  // 消息队列
|               00000000000000000000           // 消息队列文件
```
### (4). index文件
index索引文件,其主要设计理念就是为了加速消息的检索功能,根据消息的属性快速从commit文件中检索消息.

```
// index索引目录结构
\---index                
        20191205095426975                      // 索引文件
```
