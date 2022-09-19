---
layout: post
title: 'Zeebe Gateway源码之StandaloneGateway创建BrokerClient详解(三)' 
date: 2022-09-16
author: 李新
tags:  Zeebe
---

### (1). 概述
在这一节,仍然把:StandaloneGateway的创建过程给剖析完毕,这一小节,主要剖析:BrokerClient.

### (2). StandaloneGateway.createBrokerClient
```
private BrokerClient createBrokerClient(final GatewayCfg config) {
	return new BrokerClientImpl(
		config,
		// *********************************************************
		// MessagingService是atomix基于Netty组件之上,进行一消息处理器
		// *********************************************************
		atomixCluster.getMessagingService(),
		// *********************************************************
		// ClusterMembershipService主要用于获取成员列表,呆会会扔出相关的API出来
		// *********************************************************
		atomixCluster.getMembershipService(),
		// *********************************************************
		// ClusterEventService是集群事件服务,是atomix为集群的机器提供通信的一个组件,主要用于广播以及订阅信息
		// *********************************************************
		atomixCluster.getEventService(),
		// *********************************************************
		// ActorScheduler在上一小节剖析过了,他主要是两组线程池(CPU密集型和IO密集型)
		// *********************************************************
		actorScheduler,
		false);
}
```
### (3). BrokerClient接口
```
public interface BrokerClient extends AutoCloseable {
	// 发送请求
	<T> CompletableFuture<BrokerResponse<T>> sendRequest(BrokerRequest<T> request);
	
	// 发送请求,并附加上超时
	<T> CompletableFuture<BrokerResponse<T>> sendRequest(BrokerRequest<T> request, Duration requestTimeout);
	
	// 发送请求,并重试
	<T> CompletableFuture<BrokerResponse<T>> sendRequestWithRetry(BrokerRequest<T> request);
	
	// 发送请求,并重试,并附加上超时
	<T> CompletableFuture<BrokerResponse<T>> sendRequestWithRetry(BrokerRequest<T> request, Duration requestTimeout);
	
	<T> void sendRequestWithRetry(BrokerRequest<T> request,BrokerResponseConsumer<T> responseConsumer,Consumer<Throwable> throwableConsumer);
	
	// 获取Broker集群信息
	BrokerTopologyManager getTopologyManager();
	
	// 没太看明白这是啥 
	void subscribeJobAvailableNotification(String topic, Consumer<String> handler);
}	
```
### (4). 总结
根据BrokerClient接口的能力,就能知道这个类主要用于向Broker发送请求来着的,具体的子类是如何实现的,暂时不太关心,因为,我的主要任务是:找到Gateway(Netty解码[GRPC对HTTP2进行解码的关键])接收请求,转发请求给Broker才是关键. 
