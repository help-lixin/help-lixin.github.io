---
layout: post
title: 'Flume TailDir'
date: 2020-11-11
author: 李新
tags: Flume
---

### (1). 监控多个目录或文件,并能实现断点续传功能
> 1. 监控多个目录    
> 2. 断点续传(exec不支持)

### (2). 创建配置(taildir-flume-file.conf)
```
a1.sources = r1
a1.channels = c1
a1.sinks = k1

a1.sources.r1.type = TAILDIR

# taildir记录每个文件最后读取的position
a1.sources.r1.positionFile=/tmp/log/flume/taildir_position.json
# 定义多个group
a1.sources.r1.filegroups=f1 f2

# 配置单个group(f1)
a1.sources.r1.filegroups.f1=/Users/lixin/Workspace/spring-cloud-sample-provider/logs/access_log.2020-11-12.log
a1.sources.r1.headers.f1.headerKey1=value1

# 配置单个group(f2)
# 只监听以*.log结尾的文件
a1.sources.r1.filegroups.f2=/Users/lixin/Workspace/spring-cloud-sample-provider/logs/level-logs/.*.log
a1.sources.r1.headers.f2.headerKey1 = value2
a1.sources.r1.headers.f2.headerKey2 = value2-2

a1.sources.r1.fileHeader = true
a1.sources.ri.maxBatchCount = 1000
a1.sources.r1.channels = c1

# Use a channel which buffers events in memory
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100


a1.sinks.k1.type=file_roll
a1.sinks.k1.sink.rollInterval=0
a1.sinks.k1.sink.directory=/tmp/log/flume
a1.sinks.k1.channel=c1
```

### (3). 启动agent
```
bin/flume-ng agent --name a1 --conf conf --conf-file ./works/taildir-flume-file.conf  -Dflume.root.logger=INFO,console
```