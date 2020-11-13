---
layout: post
title: 'Flume Interceptor'
date: 2020-11-11
author: 李新
tags: Flume
---

### (1). 需求

> 自定义拦截器 

### (2). 添加依赖
```
<dependency>
    <groupId>org.apache.flume</groupId>
    <artifactId>flume-ng-core</artifactId>
    <version>1.8.0</version>
    <!-- 仅仅在编译时需要 -->
    <scope>compile</scope>
</dependency>
```
### (3). 创建Interceptor
```
package help.lixin.flume.interceptor;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;

import org.apache.flume.Context;
import org.apache.flume.Event;
import org.apache.flume.interceptor.Interceptor;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * 自定义拦截器
 * 
 * @author lixin
 */
public class CustomerInterceptor implements Interceptor {
	private static final Logger logger = LoggerFactory.getLogger(CustomerInterceptor.class);
	private static final String LEVEL = "LEVEL";
	private static final String DEBUG = "DEBUG";
	private static final String ERROR = "ERROR";
	private static final String INFO = "INFO";
	private static final String WARN = "WARN";

	private List<Event> tmpEvents;

	public void initialize() {
		tmpEvents = new ArrayList<Event>();
	}

	public Event intercept(Event event) {
		Map<String, String> headers = event.getHeaders();
		String body = new String(event.getBody());
        // 在Header中添加信息.
		if (body.contains("DEBUG")) {
			headers.put(LEVEL, DEBUG);
		} else if (body.contains("ERROR")) {
			headers.put(LEVEL, ERROR);
		} else if (body.contains("INFO")) {
			headers.put(LEVEL, INFO);
		} else if (body.contains("WARN")) {
			headers.put(LEVEL, WARN);
		}
		return event;
	}

	public List<Event> intercept(List<Event> events) {
		tmpEvents.clear();
		for (Event event : events) {
			tmpEvents.add(intercept(event));
		}
		return tmpEvents;
	}

	public void close() {
	}

	public static class Builder implements Interceptor.Builder {
		public void configure(Context context) {
		}

		public Interceptor build() {
			return new CustomerInterceptor();
		}
	}
}
```
### (4). 打包

> 将Maven项目打包成jar,并添加到${FLUME}/lib/目录下

### (5). Flum1配置(taildir-flume-customer-interceptor.conf)
```
a1.sources = r1
a1.channels = c1 c2
a1.sinks = k1 k2

# 定义source的selector类型
a1.sources.r1.selector.type=multiplexing
a1.sources.r1.selector.header=LEVEL
a1.sources.r1.selector.mapping.DEBUG = c1
a1.sources.r1.selector.mapping.ERROR = c1
a1.sources.r1.selector.mapping.INFO  = c2 
a1.sources.r1.selector.mapping.WARN  = c2 

a1.sources.r1.type=TAILDIR
a1.sources.r1.positionFile=/tmp/log/flume/taildir_position.json
# define groups
a1.sources.r1.filegroups=f2
#group f2
a1.sources.r1.filegroups.f2=/Users/lixin/Workspace/spring-cloud-sample-provider/logs/level-logs/.*.log
a1.sources.r1.fileHeader = true
a1.sources.ri.maxBatchCount = 1000
a1.sources.r1.interceptors = custimerInceptor
a1.sources.r1.interceptors.custimerInceptor.type=help.lixin.flume.interceptor.CustomerInterceptor$Builder
a1.sources.r1.channels=c1 c2

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
### (6). Flume2配置(avro-flume-log.conf)
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
### (7). Flume2配置(avro-flume-file.conf)
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
### (8). Flume1启动
```
bin/flume-ng agent --name a1 --conf conf --conf-file ./works/taildir-flume-customer-interceptor.conf  -Dflume.root.logger=INFO,console
```
### (9). Flume2启动
```
bin/flume-ng agent --name a1 --conf conf --conf-file ./works/avro-flume-log.conf  -Dflume.root.logger=INFO,console
```
### (10). Flume3启动
```
bin/flume-ng agent --name a1 --conf conf --conf-file ./works/avro-flume-file.conf  -Dflume.root.logger=INFO,console
```
### (10). 总结
> ERROR/DEBUG级别日志会在Flume(4545端口)的**控制台**输出    
> INFO/WARN级别日志会在Flume(4546端口)的**文件目录(/tmp/log/flume)**里    