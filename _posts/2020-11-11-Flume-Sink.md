---
layout: post
title: 'Flume 自定义Sink'
date: 2020-11-11
author: 李新
tags: Flume
---

### (1). 需求

> 自定义Sink 

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
### (4). CustomerSink
```
package help.lixin.flume.sink;

import java.util.concurrent.TimeUnit;

import org.apache.flume.Channel;
import org.apache.flume.Context;
import org.apache.flume.Event;
import org.apache.flume.EventDeliveryException;
import org.apache.flume.Transaction;
import org.apache.flume.conf.Configurable;
import org.apache.flume.sink.AbstractSink;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class CustomerSink extends AbstractSink implements Configurable {

	private static final Logger logger = LoggerFactory.getLogger(CustomerSink.class);

	// 模板前缀
	private String templatePrefix;
	// 模板后缀
	private String templateSuffix;

	@Override
	public Status process() throws EventDeliveryException {
		Status status = null;
		Channel channel = getChannel();
		Transaction tx = channel.getTransaction();
		tx.begin();
		try {
			Event event;
			while (true) {
				event = channel.take();
				if (null == event) {
					TimeUnit.SECONDS.sleep(1);
				}
				
				if (null != event) {
					break;
				}
			} // end while
			logger.info(templatePrefix + "--" + new String(event.getBody()) + "--" + templateSuffix);
			tx.commit();
			status = Status.READY;
		} catch (Throwable t) {
			logger.error("customer sink error:{}", t);
			tx.rollback();
			status = Status.BACKOFF;
		} finally {
			if (null != tx) {
				tx.close();
			}
		}
		return status;
	}

	static class CustomerSinkConstants {
		private static final String TEMPLATE_PREFIX = "prefix";
		private static final String TEMPLATE_SUFFIX = "suffix";
	}

	@Override
	public void configure(Context context) {
		this.templatePrefix = context.getString(CustomerSinkConstants.TEMPLATE_PREFIX, "TEST:");
		this.templateSuffix = context.getString(CustomerSinkConstants.TEMPLATE_SUFFIX, ":TEST");
	}
}

```
### (5). 打包

> 将Maven项目打包成jar,并添加到${FLUME}/lib/目录下

### (6). 创建配置(customer-source-sink.conf)
```
a1.sources=r1
a1.channels=c1
a1.sinks=k1


# 配置自定义Source
a1.sources.r1.type=help.lixin.flume.source.CustomerSource
# 配置相关属性
a1.sources.r1.prefix=HELLO
a1.sources.r1.suffix=WORLD
a1.sources.r1.maxBytesToLog=200
# Bind the source and sink to the channel
a1.sources.r1.channels=c1

# Use a channel which buffers events in memory
a1.channels.c1.type=memory
a1.channels.c1.capacity=1000
a1.channels.c1.transactionCapacity=100


# Describe the sink
a1.sinks.k1.type=help.lixin.flume.sink.CustomerSink
a1.sinks.k1.prefix=TEST-PREFIX
a1.skins.k1.suffix=TEST-SUFFIX
# Bind the source and sink to the channel
a1.sinks.k1.channel=c1
```
### (7). 启动agent
```
 bin/flume-ng agent --name a1 --conf conf --conf-file ./works/customer-source-sink.conf  -Dflume.root.logger=INFO,console
```