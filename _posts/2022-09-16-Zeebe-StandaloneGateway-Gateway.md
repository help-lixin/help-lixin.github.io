---
layout: post
title: 'Zeebe Gateway源码之StandaloneGateway创建Gateway详解(四)' 
date: 2022-09-16
author: 李新
tags:  Zeebe
---

### (1). 概述
在前面对BrokerClient进行了剖析,它相当于Client SDK,用于向Broker发送请求,在这里分析比较核心的类:Gateway.

### (2). StandaloneGateway.run
```
Gateway gateway = new Gateway(
         configuration, 
		 // 1. 传递了一个Function,在使用时再调用.
		 this::createBrokerClient, 
		 actorScheduler);
```
### (3). Gateway构造器
```
public final class Gateway {
		
	// 2. 默认Server构造工厂	
	private static final Function<GatewayCfg, ServerBuilder> DEFAULT_SERVER_BUILDER_FACTORY = cfg -> setNetworkConfig(cfg.getNetwork());

	public Gateway(
		  final GatewayCfg gatewayCfg,
		  final Function<GatewayCfg, BrokerClient> brokerClientFactory,
		  final ActorSchedulingService actorSchedulingService) {
		// ************************************************************************	  
		// 1. 设置一个默认构建:io.grpc.Server类的工厂.
		// ************************************************************************	  
		this(gatewayCfg, brokerClientFactory, DEFAULT_SERVER_BUILDER_FACTORY, actorSchedulingService);
	}// end Gateway
	
	
	private static NettyServerBuilder setNetworkConfig(final NetworkCfg cfg) {
		final Duration minKeepAliveInterval = cfg.getMinKeepAliveInterval();

		if (minKeepAliveInterval.isNegative() || minKeepAliveInterval.isZero()) {
		  throw new IllegalArgumentException("Minimum keep alive interval must be positive.");
		}

		// **************************************************************
		// 也就是说:Gateway监听的是26500端口
        // gateway:26500
		// **************************************************************
		return NettyServerBuilder.forAddress(new InetSocketAddress(cfg.getHost(), cfg.getPort()))
			.permitKeepAliveTime(minKeepAliveInterval.toMillis(), TimeUnit.MILLISECONDS)
			.permitKeepAliveWithoutCalls(false);
    } // end setNetworkConfig
}
```
### (4). 

### (5). 

### (6). 

### (7). 

### (8). 

### (9). 

### (10). 

### (11). 

### (12). 