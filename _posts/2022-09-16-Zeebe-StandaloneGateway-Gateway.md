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
```
public void start() throws IOException {
    healthManager.setStatus(Status.STARTING);
    brokerClient = buildBrokerClient();

    final ActivateJobsHandler activateJobsHandler;
	// 判断是否开启了轮询
    if (gatewayCfg.getLongPolling().isEnabled()) { // true
      final LongPollingActivateJobsHandler longPollingHandler = buildLongPollingHandler(brokerClient);
      actorSchedulingService.submitActor(longPollingHandler);
      activateJobsHandler = longPollingHandler;
    } else {
      activateJobsHandler = new RoundRobinActivateJobsHandler(brokerClient);
    }

    // *******************************************************************************
	// 1. 实际上分析到这里就大概知道了,Netty(GRPC)接收请求后,会把请求通过:EndpointManager进行转发.
    // EndpointManager类内部持有BrokerClient,通过BrokerClient与Broker进行通信来着的.
	// 比如:completeJob/cancelProcessInstance/createProcessInstance/createProcessInstanceWithResult
	// *******************************************************************************
    final EndpointManager endpointManager = new EndpointManager(brokerClient, activateJobsHandler);
	
	// *******************************************************************************
	// 2. GatewayGrpcService包裹了EndpointManager,代表在EndpointManager的方法上进行了增强.
	// *******************************************************************************
    final GatewayGrpcService gatewayGrpcService = new GatewayGrpcService(endpointManager);
    final ServerBuilder<?> serverBuilder = serverBuilderFactory.apply(gatewayCfg);

    final SecurityCfg securityCfg = gatewayCfg.getSecurity();
    if (securityCfg.isEnabled()) {
      setSecurityConfig(serverBuilder, securityCfg);
    }

    server =
        serverBuilder
            .addService(applyInterceptors(gatewayGrpcService))
            .addService(
                ServerInterceptors.intercept(
                    healthManager.getHealthService(), MONITORING_SERVER_INTERCEPTOR))
            .build();
			
    server.start();
    healthManager.setStatus(Status.RUNNING);
} // end 
```
### (5). 总结
> Gateway启动时,会绑定三个端口:  
+ 9600(web端口,用于常用的监控数据).  
+ 26500(Gateway对外提供的端口,主要用于JobWoker的通信).  
+ 26502(Gateway与Broker进行通信的端口,即Atomix集群监听端口).   
