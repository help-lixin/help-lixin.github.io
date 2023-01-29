---
layout: post
title: 'Pulsar源码之Producer分析中篇(四)' 
date: 2023-01-22
author: 李新
tags:  Pulsar
---

### (1). 概述
前面对Producer进行了剖析,发现,最后,它却委托给了:PulsarClientImpl处理,所以,这一小篇,主要剖析:PulsarClientImpl创建Producer的过程.  

### (2). PulsarClientImpl.createProducerAsync
```
public <T> CompletableFuture<Producer<T>> createProducerAsync(
		//  生产者配置
		ProducerConfigurationData conf, 
		Schema<T> schema, 
		// 生产者责任链的装饰器类
		ProducerInterceptors interceptors) {
	
	if (conf == null) {
		return FutureUtil.failedFuture(new PulsarClientException.InvalidConfigurationException("Producer configuration undefined"));
	}

	if (schema instanceof AutoConsumeSchema) {
		return FutureUtil.failedFuture(new PulsarClientException.InvalidConfigurationException("AutoConsumeSchema is only used by consumers to detect schemas automatically"));
	}

	// 验证状态
	if (state.get() != State.Open) {
		return FutureUtil.failedFuture(new PulsarClientException.AlreadyClosedException("Client already closed : state = " + state.get()));
	}

	// 主题
	String topic = conf.getTopicName();

	// 验证下topic
	if (!TopicName.isValid(topic)) {
		return FutureUtil.failedFuture(new PulsarClientException.InvalidTopicNameException("Invalid topic name: '" + topic + "'"));
	}

	// 针对:AutoProduceBytesSchema进行处理
	if (schema instanceof AutoProduceBytesSchema) {
		// 这一部份的内容,咱先略过,先剖析过主流,我这里的Schema是String
		AutoProduceBytesSchema autoProduceBytesSchema = (AutoProduceBytesSchema) schema;
		if (autoProduceBytesSchema.schemaInitialized()) {
			return createProducerAsync(topic, conf, schema, interceptors);
		}
		return lookup.getSchema(TopicName.get(conf.getTopicName()))
				.thenCompose(schemaInfoOptional -> {
					if (schemaInfoOptional.isPresent()) {
						autoProduceBytesSchema.setSchema(Schema.getSchema(schemaInfoOptional.get()));
					} else {
						autoProduceBytesSchema.setSchema(Schema.BYTES);
					}
					
					// ****************************************************************
					// 最终委托给:createProducerAsync方法
					// ****************************************************************
					return createProducerAsync(topic, conf, schema, interceptors);
				});
	} else {
		// ****************************************************************
		// 最终委托给:createProducerAsync方法
		// ****************************************************************
		return createProducerAsync(topic, conf, schema, interceptors);
	}
} // end 
```
### (3). PulsarClientImpl.createProducerAsync
```
private <T> CompletableFuture<Producer<T>> createProducerAsync(String topic,
                                                                   ProducerConfigurationData conf,
                                                                   Schema<T> schema,
                                                                   ProducerInterceptors interceptors) {
	CompletableFuture<Producer<T>> producerCreatedFuture = new CompletableFuture<>();
	
	// ************************************************************************************************
	// getPartitionedTopicMetadata是向Pulsar服务端发起请求,获得Topic的信息(比如:partition的数量)
	// ************************************************************************************************
	getPartitionedTopicMetadata(topic)
	.thenAccept(metadata -> {
		
		if (log.isDebugEnabled()) {
			log.debug("[{}] Received topic metadata. partitions: {}", topic, metadata.partitions);
		}

		// ************************************************************************************************
		// 如果topic只有一个分区,则创建的是实现类是: ProducerImpl
		// 如果topic只有多个分区,则创建的是实现类是: PartitionedProducerImpl
		// ************************************************************************************************
		ProducerBase<T> producer;
		if (metadata.partitions > 0) {
			producer = newPartitionedProducerImpl(topic, conf, schema, interceptors, producerCreatedFuture,metadata);
		} else {
			producer = newProducerImpl(topic, -1, conf, schema, interceptors, producerCreatedFuture);
		}
		
		// ************************************************************************************************
		// PulsarClientImpl持有创建的所有:producer对象.
		// ************************************************************************************************
		producers.add(producer);
	}).exceptionally(ex -> {
		log.warn("[{}] Failed to get partitioned topic metadata: {}", topic, ex.getMessage());
		producerCreatedFuture.completeExceptionally(ex);
		return null;
	});
	return producerCreatedFuture;
} // end createProducerAsync
```
### (4). 设计模式
> 在这里用到了策略模式,根据topic获得的信息中,根据:分区数(partitions)创建不同的:ProducerBase. 

### (5). 总结
> 1. getPartitionedTopicMetadata方法会向Pulsar Broker发起请求,获取:topic的元数据.  
> 2. 根据topic的元数据(partitions),创建不同的:ProducerBase实现类.  
>  Pulsar的做法与其它的MQ不同,Pulsar在发送数据之前,需要一个非常确定的信息(比如:tenant/namespace/topic),通过这些信息,获取到元数据之后,为topic每一个partition创建一个对应的类进行处理,优点:职责明确,各有各的缓存区来着的,缺点:对象还是挺多的.  