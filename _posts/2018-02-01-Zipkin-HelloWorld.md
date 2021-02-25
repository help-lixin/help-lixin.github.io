---
layout: post
title: 'Zipkin Hello World(一)'
date: 2018-02-01
author: 李新
tags: Zipkin
---

### (1). Zipkin是什么?
> Zipkin是Twitter的一个开源项目,允许开发者收集各个服务上的监控数据,并提供查询接口.

### (2). zipkin案例步骤
> 1. 启动zipkin server.    
> 2. 编写并运行HelloWorldTest.  
> 3. zipkin server(UI界面)查看结果.   

### (3). HelloWorldTest
```
package help.lixin.zipkin;

import java.util.UUID;
import java.util.concurrent.TimeUnit;

import brave.Span;
import brave.Tracer;
import brave.Tracing;
import brave.sampler.Sampler;
import zipkin2.reporter.AsyncReporter;
import zipkin2.reporter.okhttp3.OkHttpSender;

public class HelloWorldTest {
	public static void main(String[] args) throws Exception {
		String endpoint = "http://127.0.0.1:9411/api/v2/spans";
		OkHttpSender sender = OkHttpSender.create(endpoint);
		AsyncReporter reporter = AsyncReporter.create(sender);
		
		
		Tracing tracing = Tracing.newBuilder()
				.localServiceName("brave-service")
				.spanReporter(reporter)
				.sampler(Sampler.ALWAYS_SAMPLE)
				.build();
		
		// 创建一个trace
		Tracer tracer = tracing.tracer();
		
		// 创建跨度
		Span span = tracer.newTrace()
				//tag 用来存放业务数据,方便进行检索
				.tag("request-id", UUID.randomUUID().toString())
				.name("brave-span")
				.start();
		
		// 业务处理所消耗时间
		TimeUnit.MILLISECONDS.sleep(10);
		
		// 跨度完成
		span.finish();
		
				
		tracing.close();
		reporter.close();
		sender.close();
	}
}
```
### (7). 总结
> 内部是如何实现的,下一小节再详细分析.