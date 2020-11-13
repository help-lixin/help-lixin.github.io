---
layout: post
title: 'Flume Replicating Channel Selector'
date: 2020-11-11
author: 李新
tags: Flume
---

### (1). 业务需求
> Flum1 收集数据 传输给 Flum2 和 Flum3,要求:<font color='red'>Flum2和Flum3的数据一模一样.</font>

!["Flume 单数据源多出口"](/assets/flume/imgs/flume-multiplexing-flow.jpg)

### (2). Flume1配置(sing-source-mutil-sink.conf)
```
a1.sources = r1
a1.channels = c1 c2
a1.sinks = k1 k2

# 定义source selector为复制策略
a1.sources.r1.selector.type = replicating
a1.sources.r1.type = TAILDIR
a1.sources.r1.positionFile=/tmp/log/flume/taildir_position.json
a1.sources.r1.filegroups=f1
a1.sources.r1.filegroups.f1=/Users/lixin/Workspace/spring-cloud-sample-provider/logs/access_log.2020-11-12.log
a1.sources.r1.headers.f1.headerKey1=value1
a1.sources.r1.fileHeader = true
a1.sources.ri.maxBatchCount = 1000
# 绑定到c1和c2两个channel
a1.sources.r1.channels = c1 c2

# Use a channel which buffers events in memory
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100

a1.channels.c2.type = memory
a1.channels.c2.capacity = 1000
a1.channels.c2.transactionCapacity = 100

# k1输出到4545端口
a1.sinks.k1.type=avro
a1.sinks.k1.hostname=localhost
a1.sinks.k1.port=4545
a1.sinks.k1.channel=c1

# k2输出到4546端口
a1.sinks.k2.type=avro
a1.sinks.k2.hostname=localhost
a1.sinks.k2.port=4546
a1.sinks.k2.channel=c2
```
### (3). Flum2(avro-flume-log.conf)
```
a1.sources = r1
a1.channels = c1
a1.sinks = k1

# 从4545端口接受数据
# Describe/configure the source
a1.sources.r1.type = avro
a1.sources.r1.bind = localhost
a1.sources.r1.port = 4545
# Bind the source and sink to the channel
a1.sources.r1.channels = c1

# Use a channel which buffers events in memory
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100

# 在日志里打印
# Describe the sink
a1.sinks.k1.type = logger
# Bind the source and sink to the channel
a1.sinks.k1.channel = c1
```
### (4). Flume3(avro-flume-file.conf)
```
a1.sources = r1
a1.channels = c1
a1.sinks = k1

# 从4546端口接受数据
# Describe/configure the source
a1.sources.r1.type = avro
a1.sources.r1.bind = localhost
a1.sources.r1.port = 4546
# Bind the source and sink to the channel
a1.sources.r1.channels = c1

# Use a channel which buffers events in memory
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100


# 输出到文件里
# Describe the sink
a1.sinks.k1.type = file_roll
a1.sinks.k1.sink.rollInterval=0
a1.sinks.k1.sink.directory = /tmp/log/flume
# Bind the source and sink to the channel
a1.sinks.k1.channel = c1
```

### (5). Flume1启动agent(/works/sing-source-mutil-sink.conf)
```
bin/flume-ng agent --name a1 --conf conf --conf-file ./works/sing-source-mutil-sink.conf  -Dflume.root.logger=INFO,console
```

### (6). Flume2启动agent(/works/avro-flume-log.conf)
```
bin/flume-ng agent --name a1 --conf conf --conf-file ./works/avro-flume-log.conf  -Dflume.root.logger=INFO,console
```
### (7). Flume2启动agent(/works/avro-flume-file.conf)
```
bin/flume-ng agent --name a1 --conf conf --conf-file ./works/avro-flume-file.conf  -Dflume.root.logger=INFO,console
```
