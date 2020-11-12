---
layout: post
title: 'Flume Tomcat AccessLog'
date: 2020-11-12
author: 李新
tags: Flume
---

### (1). 监控Tomcat AccessLog
> 1. 监控Tomcat AccessLog   
> 2. 输出到另一个日志文件中    

### (2). 创建Flume配置文件(works/tomcat-accesslog-flume-logger.conf)
```
a1.sources = r1
a1.channels = c1
a1.sinks = k1

a1.sources.r1.type = exec
a1.sources.r1.command = tail -F /Users/lixin/Workspace/spring-cloud-sample-provider/logs/access_log.2020-11-12.log
a1.sources.r1.channels = c1

# Use a channel which buffers events in memory
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100

# Describe the sink
# a1.sinks.k1.type = logger
# Bind the source and sink to the channel
# a1.sinks.k1.channel = c1

a1.sinks.k1.type=file_roll
a1.sinks.k1.sink.rollInterval=0
a1.sinks.k1.sink.directory=/tmp/log/flume
a1.sinks.k1.channel=c1
```
### (3). 启动agent
```
bin/flume-ng agent --name a1 --conf conf --conf-file ./works/tomcat-accesslog-flume-logger.conf -Dflume.root.logger=INFO,console
```
