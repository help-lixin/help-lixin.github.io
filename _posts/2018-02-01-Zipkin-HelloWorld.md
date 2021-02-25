---
layout: post
title: 'Zipkin Hello World(一)'
date: 2018-02-01
author: 李新
tags: Zipkin
---

### (1). Zipkin是什么?
> Zipkin是Twitter的一个开源项目,允许开发者收集各个服务上的监控数据,并提供查询接口.

### (2). Zipkin架构图
!["zipkin architecture"](/assets/zipkin/imgs/zipkin-architecture.png)

> Instrumented server(Reporter):可以理解为agent,它和业务进程捆绑在一起.    
> Transport:传输层.     
> Collector:收集器(可以理解为Controller).  
> Store:存储层(MySQL/ES/...).    
> API: 提供接口给外部(UI)访问.   
> 总结:Reporter通过Transport向Collector(Zipkin Server)汇报数据,然后,Store存储.而API+UI负责做展现.  

### (3). ZipKin与Java整合
> 1. 通过zipkin(zipkin-reporter/zipkin-sender-okhttp3),向zipkin server汇报数据.   
> 2. 通过barve(brave),向zipkin server汇报数据.   
> 3. 通过spring-cloud-starter-sleuth,向zipkin server汇报数据.   
> spring-cloud-starter-sleuth对barve进行了整合.而brave又对zipkin进行了API的包装和简洁.   

### (4). 研究步骤
> 1. brave.   
> 2. spring-cloud-starter-sleuth.   
> 3. 原因:先了解了API,对于:spring-cloud-starter-sleuth只是一个整合而已.  
### (5). HelloWorldTest
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
				//tag 用来存放业务数据,方便在UI进行检索.
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
### (6). zipkin类图和执行流程图
!["brave类图"](/assets/zipkin/imgs/zipkin-Class-Diagram.jpg)
!["brave执行流程图"](/assets/zipkin/imgs/zipkin-Sequence-Diagram.jpg)

### (7). 总结
> 从执行流程图来看,好像也没什么好讲的啦!就是那么so easy. 