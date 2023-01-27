---
layout: post
title: 'Pulsar源码之PulsarClient构建过程分析(二)' 
date: 2023-01-22
author: 李新
tags:  Pulsar
---

### (1). 概述
这一篇,我们主要剖析PulsarClient构建过程.  

### (2). PulsarClient 构建过程
> 在这里典型的用到了Builder模式. 

```
PulsarClient client = PulsarClient.builder()
```

### (2). PulsarClient.builder
> 与我们平时的Builder方法不同,Pulsar为Client的构建过程,也独立配置了一个接口(ClientBuilder). 

```
static ClientBuilder builder() {
	return DefaultImplementation.getDefaultImplementation().newClientBuilder();
}
```
### (3). DefaultImplementation.getDefaultImplementation
> 典型的单例模式.

```
public class DefaultImplementation {
	
	private static final PulsarClientImplementationBinding IMPLEMENTATION;
	
	// ************************************************************************************
	// 单例模式
	// ************************************************************************************
	static {
		PulsarClientImplementationBinding impl = null;
		try {
			// *****************************************************************************
			// 通过反射创建:PulsarClientImplementationBinding
			// *****************************************************************************
			impl = (PulsarClientImplementationBinding) ReflectionUtils.newClassInstance("org.apache.pulsar.client.impl.PulsarClientImplementationBindingImpl").getConstructor().newInstance();
		} catch (Throwable error) {
			throw new RuntimeException("Cannot load Pulsar Client Implementation: "+error, error);
		}
		IMPLEMENTATION = impl;
	}
	
	
	public static PulsarClientImplementationBinding getDefaultImplementation() {
		return IMPLEMENTATION;
	}

}
```

### (4). PulsarClientImplementationBinding.newClientBuilder
```
public ClientBuilder newClientBuilder() {
	return new ClientBuilderImpl();
}
```
### (5). ClientBuilder.build
> 可以理解:ClientBuilder的所有的配置方法,都是对:ClientConfigurationData进行操作.  

```
public class ClientBuilderImpl implements ClientBuilder {
    // *****************************************************************
	// client配置
	// *****************************************************************
	ClientConfigurationData conf;
	
	public PulsarClient build() throws PulsarClientException {
		
		if (StringUtils.isBlank(this.conf.getServiceUrl()) && this.conf.getServiceUrlProvider() == null) {
			throw new IllegalArgumentException("service URL or service URL provider needs to be specified on the ClientBuilder object.");
		} else if (StringUtils.isNotBlank(this.conf.getServiceUrl()) && this.conf.getServiceUrlProvider() != null) {
			throw new IllegalArgumentException("Can only chose one way service URL or service URL provider.");
		} else {
			if (this.conf.getServiceUrlProvider() != null) {
				if (StringUtils.isBlank(this.conf.getServiceUrlProvider().getServiceUrl())) {
					throw new IllegalArgumentException("Cannot get service url from service url provider.");
				}

				this.conf.setServiceUrl(this.conf.getServiceUrlProvider().getServiceUrl());
			}
			
			// ***************************************************************************
			// 创建:PulsarClient
			// ***************************************************************************
			PulsarClient client = new PulsarClientImpl(this.conf);
			if (this.conf.getServiceUrlProvider() != null) {
				this.conf.getServiceUrlProvider().initialize(client);
			}

			return client;
		}
	}
	
}
```
### (6). PulsarClient UML图解
!["PulsarClient UML图解"](/assets/pulsar/imgs/PulsarClient-ClassDiagram.jpg)

### (7). 看下PulsarClient接口
> 最终要把焦点放在:PulsarClient接口上,因为,ClientBuilder只是构建过程需要的参数配置而已.  

```
public interface PulsarClient extends Closeable {
	
	// *******************************************************************
	// 生产者Builder
	// *******************************************************************
    <T> ProducerBuilder<T> newProducer(Schema<T> schema);

	// *******************************************************************
	// 消费者Builder
	// *******************************************************************
    <T> ConsumerBuilder<T> newConsumer(Schema<T> schema);
	
	// *******************************************************************
	// 读消息Builder
	// *******************************************************************
    <T> ReaderBuilder<T> newReader(Schema<T> schema);

	// *******************************************************************
	// 配置serviceUrl
	// *******************************************************************
    void updateServiceUrl(String serviceUrl) throws PulsarClientException;

	// *******************************************************************
	// 根据topic获取分区信息
	// *******************************************************************
    CompletableFuture<List<String>> getPartitionsForTopic(String topic);

	// *******************************************************************
	// 事务Builder
	// *******************************************************************
    TransactionBuilder newTransaction() throws PulsarClientException;
}
```
### (8). 设计模式
> 在这几行简单的代码里,用到了哪些设计模式?   
> 构建者模式   
> 单例模式   
### (9). 总结
