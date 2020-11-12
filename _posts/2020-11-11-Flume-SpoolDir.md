---
layout: post
title: 'Flume SpoolDir'
date: 2020-11-11
author: 李新
tags: Flume
---

### (1). 监控某个目录,并输出结果到文件中
> 1. 监控目录
> 2. 输出到某个文件中
### (2). 创建Flume配置(dir-flume-file.conf)
```
a1.sources = r1
a1.channels = c1
a1.sinks = k1

# 监控目录
a1.sources.r1.type = spooldir
a1.sources.r1.spoolDir = /tmp/logs
a1.sources.r1.fileHeader = true
a1.sources.r1.channels = c1

# Use a channel which buffers events in memory
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100

# Describe the sink
# a1.sinks.k1.type = logger
# Bind the source and sink to the channel
# a1.sinks.k1.channel = c1

# 把结果输出到文件里
a1.sinks.k1.type=file_roll
a1.sinks.k1.sink.rollInterval=0
a1.sinks.k1.sink.directory=/tmp/log/flume
a1.sinks.k1.channel=c1
```
### (3). 启动Agent
```
bin/flume-ng agent --name a1 --conf conf --conf-file ./works/dir-flume-file.conf  -Dflume.root.logger=INFO,console
```
