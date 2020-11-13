---
layout: post
title: 'Flume Sink Load Balancing'
date: 2020-11-11
author: 李新
tags: Flume
---

### (1). 需求
> Flume1接受数据,同步给Flum2和Flume3.<font color='red'>同步时,如果出现错误,需要进行失败转移或负载均衡.</font>

!["Flume 单数据源多出口"](/assets/flume/imgs/flume-multiplexing-flow.jpg)

### (2). Flume1配置(sink-load-balance.conf)
```
a1.sources = r1
a1.channels = c1
a1.sinks = k1 k2


# define sink group 
a1.sinkgroups = g1
a1.sinkgroups.g1.sinks = k1 k2
a1.sinkgroups.g1.processor.type = load_balance

# 定义失败转移或负载均衡
# a1.sinkgroups.g1.processor.type = failover
# a1.sinkgroups.g1.processor.priority.k1 = 5
# a1.sinkgroups.g1.processor.priority.k2 = 10
# a1.sinkgroups.g1.processor.maxpenalty = 10000


a1.sources.r1.type = TAILDIR
a1.sources.r1.positionFile=/tmp/log/flume/taildir_position.json
# define groups
a1.sources.r1.filegroups=f1
a1.sources.r1.filegroups.f1=/Users/lixin/Workspace/spring-cloud-sample-provider/logs/.*access_log.*
a1.sources.r1.headers.f1.headerKey1=value1
a1.sources.r1.fileHeader = true
a1.sources.ri.maxBatchCount = 1000
a1.sources.r1.channels = c1


# Use a channel which buffers events in memory
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100


a1.sinks.k1.type=avro
a1.sinks.k1.hostname=localhost
a1.sinks.k1.port=4545
a1.sinks.k1.channel=c1


a1.sinks.k2.type=avro
a1.sinks.k2.hostname=localhost
a1.sinks.k2.port=4546
a1.sinks.k2.channel=c1
```

### (3). Flume2配置(avro-flume-log.conf)
```
a1.sources = r1
a1.channels = c1
a1.sinks = k1

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

# Describe the sink
a1.sinks.k1.type = logger
# Bind the source and sink to the channel
a1.sinks.k1.channel = c1
```

### (4). Flume3配置(avro-flume-file.conf)
```
a1.sources = r1
a1.channels = c1
a1.sinks = k1

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


# Describe the sink
a1.sinks.k1.type = file_roll
a1.sinks.k1.sink.rollInterval=0
a1.sinks.k1.sink.directory = /tmp/log/flume
# Bind the source and sink to the channel
a1.sinks.k1.channel = c1
```

### (5). Flume1启动agent
```
bin/flume-ng agent --name a1 --conf conf --conf-file ./works/sink-load-balance.conf  -Dflume.root.logger=INFO,console
```

### (6). Flume2启动agent
```
bin/flume-ng agent --name a1 --conf conf --conf-file ./works/avro-flume-log.conf  -Dflume.root.logger=INFO,console
```

### (7). Flume3启动agent
```
bin/flume-ng agent --name a1 --conf conf --conf-file ./works/avro-flume-file.conf  -Dflume.root.logger=INFO,console
```
