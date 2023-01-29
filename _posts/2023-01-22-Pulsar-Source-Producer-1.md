---
layout: post
title: 'Pulsar源码之Producer分析上篇(四)' 
date: 2023-01-22
author: 李新
tags:  Pulsar
---

### (1). 概述
在前面,简单的了解了下PulsarClient,在这里,先剖析:Producer

### (2). Producer创建过程
```
Producer<String> producer = client.newProducer(Schema.STRING)
		//
		.producerName("my-producer")
		// 指定生产者模式做测试
		// .accessMode(ProducerAccessMode.Shared)
		.topic(TOPIC)
		// 发送超时配置
		.sendTimeout(10, TimeUnit.SECONDS)
		// 
		.create();
```
### (3). PulsarClientImpl.newProducer
```
public <T> ProducerBuilder<T> newProducer(Schema<T> schema) {
	// ***********************************************************
	// 咦,居然是又创建了一个:Builder
	// ***********************************************************
	return new ProducerBuilderImpl<>(this, schema);
}
```
### (4). ProducerBuilderImpl构造器
> 对前面的PulsarClient分析时,其实也能理解了,所谓的Build实际内部是Hold住一个配置文件,然后,Build类的所有操作方法,都是把信息往配置上进行靠拢而已.   

```
public ProducerBuilderImpl(PulsarClientImpl client, Schema<T> schema) {
	// **********************************************************************
	// 委托给了另一个构造器
	// **********************************************************************
	this(client, new ProducerConfigurationData(), schema);
} // end 

private ProducerBuilderImpl(PulsarClientImpl client, ProducerConfigurationData conf, Schema<T> schema) {
	this.client = client;
	this.conf = conf;
	this.schema = schema;
} // end 
```
### (5). ProducerBuilderImpl.create
```
public Producer<T> create() throws PulsarClientException {
	try {
		// **************************************************************************
		// 委托给了:createAsync
		// **************************************************************************
		return createAsync().get();
	} catch (Exception e) {
		throw PulsarClientException.unwrap(e);
	}
} // end create
```
### (6). ProducerBuilderImpl.createAsync
```
public CompletableFuture<Producer<T>> createAsync() {
	// config validation
	checkArgument(!(conf.isBatchingEnabled() && conf.isChunkingEnabled()), "Batching and chunking of messages can't be enabled together");
	if (conf.getTopicName() == null) {
		return FutureUtil.failedFuture(new IllegalArgumentException("Topic name must be set on the producer builder"));
	}

	try {
		// ***********************************************************************************
		// 配置生产者生产消息的路由模式
		// MessageRoutingMode.SinglePartition          : 生产消息时,消息只落在Topic下某一个固定分区内.
		// MessageRoutingMode.RoundRobinPartition      : 轮询模式.
		// MessageRoutingMode.CustomPartition          : 自定义模式,该模式下,还需要配合:MessageRouter的使用.
		// ***********************************************************************************
		setMessageRoutingMode();
	} catch(PulsarClientException pce) {
		return FutureUtil.failedFuture(pce);
	}

	// ***********************************************************************************
	// 又委托给了:PulsarClientImpl.createProducerAsync进行处理
	// 其实,也能理解,在上一篇分析时,有一带而过:PulsarClient初始化时,是创建了连接池来着,里面持有网络通信的客户端来着的.  
	// ***********************************************************************************
	return interceptorList == null || interceptorList.size() == 0 ?
			client.createProducerAsync(conf, schema, null) :
			client.createProducerAsync(conf, schema, new ProducerInterceptors(interceptorList));
}
```
### (7). 设计模式
> Producer生产者构建过程用到了哪些设计模式? 
> 1. Builder模式(ProducerBuilderImpl).    
> 2. 装饰器模式(ProducerInterceptors).   
> 3. 责任链模式(ProducerInterceptor).   

### (8). 总结
我剖析到这里,就暂停,是因为,再往下剖析的话,就不属于:Producer的职责了,为了不让内容显得太过长,下一篇继续剖析完:createProducerAsync. 