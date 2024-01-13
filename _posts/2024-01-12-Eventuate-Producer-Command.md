---
layout: post
title: 'Eventuate Command Producer生产者' 
date: 2024-01-12
author: 李新
tags:  Eventuate
---

### (1). 背景

这一篇,主要剖析:生产者生产消息的过程(不剖析具体代码行),从中能发现,生产者仅仅只是依赖于DB而已. 

### (2). Eventuate Command生产者类图

![Eventuate Command生产者类图](/assets/eventuate/imgs/producer-command-send.jpg)

### (3). Eventuate Command生产者时序图

![Eventuate Command生产者时序图](/assets/eventuate/imgs/producer-command-send-sequence.jpg)

### (4). Eventuate Command生产者总结
```
Eventuate的生产者端比较简单,把消息存入到DB(eventuate.message)里即可,需要注意:这个时候并不存在,所谓的topic概念来着的. 
最终的SQL语句如下: 
insert into eventuate.message(id, destination, headers, payload, creation_time, published, message_partition) values("0000018cfcc3c8a1-66f8c7df475c0000", "commandChannel123456789", "{"PARTITION_ID":"6d3e6246-f4f9-4856-85b7-59d1a1c79275","DATE":"Fri, 12 Jan 2024 08:19:23 GMT","command_type":"help.lixin.domain.AccountAddCommand","command_reply_to":"replyChannel-123456789","DESTINATION":"commandChannel123456789","command__destination":"commandChannel123456789","ID":"0000018cfcc3c8a1-66f8c7df475c0000"}", "{"money":-564005347}", ROUND(UNIX_TIMESTAMP(CURTIME(4)) * 1000), 0, null)
```
