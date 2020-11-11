---
layout: post
title: 'Flume Hello World'
date: 2017-07-08
author: 李新
tags: Flume
---

### (1). 使用Flume监听一个端口,收集端口数据,并打印到控制台
### (2). 创建配置文件目录

> mkdir /Users/lixin/Developer/apache-flume-1.8.0-bin/works

### (3). 创建配置(netcat-flume-logger.conf)
```
a1.sources = r1
a1.channels = c1
a1.sinks = k1

# Describe/configure the source
a1.sources.r1.type = netcat
a1.sources.r1.bind = localhost
a1.sources.r1.port = 8888
# Bind the source and sink to the channel
# 一个Source可以绑定到多个Channel上
a1.sources.r1.channels = c1

# Use a channel which buffers events in memory
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100


# Describe the sink
a1.sinks.k1.type = logger
# Bind the source and sink to the channel
# 一个Sink只能有一个Channel
a1.sinks.k1.channel = c1
```

### (4). 启动agent
```
./bin/flume-ng agent --name a1  --conf conf  --conf-file ./works/netcat-flume-logger.conf -Dflume.root.logger=INFO,console

# ./bin/flume-ng agent -n a1 -c conf -f ./works/netcat-flume-logger.conf -Dflume.root.logger=INFO,console
```

>   --name         : 指定agent名称    
>   --conf         : 指定flume conf目录  
>   --conf-flie    : 指定配置文件    

!["Flume Agent启动"](/assets/flume/imgs/flume-agent-start.png)

### (5). NC测试
!["NC连接到Flume"](/assets/flume/imgs/nc-connect-flume.jpg)
!["Flume事件日志"](/assets/flume/imgs/flume-agent-event-log.png)