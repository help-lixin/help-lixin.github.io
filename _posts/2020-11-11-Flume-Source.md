---
layout: post
title: 'Flume Source'
date: 2020-11-11
author: 李新
tags: Flume
---

### (1). 需求

> 自定义Source 

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
### (4). CustomerSource
```
package help.lixin.flume.source;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.TimeUnit;

import org.apache.commons.lang.StringUtils;
import org.apache.flume.Context;
import org.apache.flume.Event;
import org.apache.flume.EventDeliveryException;
import org.apache.flume.PollableSource;
import org.apache.flume.conf.Configurable;
import org.apache.flume.event.SimpleEvent;
import org.apache.flume.source.AbstractSource;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class CustomerSource extends AbstractSource implements Configurable, PollableSource {
	private static final Logger logger = LoggerFactory.getLogger(CustomerSource.class);
	// 模板前缀
	private String templatePrefix;
	// 模板后缀
	private String templateSuffix;

	@Override
	public Status process() throws EventDeliveryException {
		Status status = null;
		try {
			// Sleep 2 Seconds
			TimeUnit.SECONDS.sleep(5);
			List<Event> events = new ArrayList<Event>();
			for (int i = 0; i < 5; i++) {
				Event event = new SimpleEvent();
				event.setBody((templatePrefix + "--" + i + "--" + templateSuffix).getBytes());
				events.add(event);
			}

			getChannelProcessor().processEventBatch(events);
			status = Status.READY;
		} catch (Throwable e) {
			logger.error("process error:{}", e);
			status = Status.BACKOFF;
		}
		return status;
	}

	@Override
	public void configure(Context context) {
		this.templatePrefix = context.getString(CustomerConstants.TEMPLATE_PREFIX, "TEST-");
		this.templateSuffix = context.getString(CustomerConstants.TEMPLATE_SUFFIX);
		if (StringUtils.isEmpty(templateSuffix)) {
			throw new IllegalArgumentException(
					"Required parameter " + CustomerConstants.TEMPLATE_SUFFIX + " must exist and may not be null");
		}
	}

	@Override
	public long getBackOffSleepIncrement() {
		return 0;
	}

	@Override
	public long getMaxBackOffSleepInterval() {
		return 0;
	}

	static class CustomerConstants {
		private static final String TEMPLATE_PREFIX = "prefix";
		private static final String TEMPLATE_SUFFIX = "suffix";
	}
}

```
### (5). 打包

> 将Maven项目打包成jar,并添加到${FLUME}/lib/目录下

### (6). 创建配置(customer-source.conf)
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
a1.sinks.k1.type=logger
# Bind the source and sink to the channel
a1.sinks.k1.channel=c1
```

### (7). 启动agent
```
 bin/flume-ng agent --name a1 --conf conf --conf-file ./works/customer-source.conf  -Dflume.root.logger=INFO,console
```
### (8). 查看结果
!["自定义Customer"](/assets/flume/imgs/customer-source.png)
