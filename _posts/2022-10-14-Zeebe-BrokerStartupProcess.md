---
layout: post
title: 'Zeebe源码之BrokerStartupProcess(二)' 
date: 2022-10-14
author: 李新
tags:  Zeebe
---

### (1). 概述
前面对Broker进行了剖析,在这里主要剖析:BrokerStartupProcess. 

### (2). BrokerStartupProcess构建器
```
public BrokerStartupProcess(final BrokerStartupContext brokerStartupContext) {
	// 并发控制器
    concurrencyControl = brokerStartupContext.getConcurrencyControl();
    context = brokerStartupContext;

    final var brokerStepMetrics = new BrokerStepMetrics();
    
	// **********************************************************
	// 1. 构建启动步骤,从变量命名上来看,就知道这是一个未经装饰的StartupStep,意味着:后面应该有装饰的StartupStep
	// **********************************************************
    final var undecoratedSteps = buildStartupSteps(brokerStartupContext.getBrokerConfiguration());


    // **********************************************************
	// 2. 通过BrokerStepMetricDecorator Wrapper前面的:StartupStep
	// **********************************************************
    final var decoratedSteps =
        undecoratedSteps.stream()
            .map(step -> new BrokerStepMetricDecorator(brokerStepMetrics, step))
            .collect(Collectors.toList());
			
    // **********************************************************
	// 3. StartupProcess包裹着所有的:StartupStep
	// **********************************************************
    startupProcess = new StartupProcess<>(LOG, decoratedSteps);
} // end 构建器
```
### (3). buildStartupSteps
> buildStartupSteps的作用主要是用来配置:StartupStep,这个类是核心类,下一篇会进行详细剖析. 

```
private List<StartupStep<BrokerStartupContext>> buildStartupSteps(final BrokerCfg config) {
    final var result = new ArrayList<StartupStep<BrokerStartupContext>>();

    if (config.getData().isDiskUsageMonitoringEnabled()) {
	  // 磁盘使用情况管理
      result.add(new DiskSpaceUsageMonitorStep());
    }
	
    result.add(new MonitoringServerStep());
	// BrokerAdmin管理
    result.add(new BrokerAdminServiceStep());

	// Atomix集群管理
    result.add(new ClusterServicesStep());
	
	// API管理
    result.add(new ApiMessagingServiceStep());
    result.add(new CommandApiServiceStep());
    result.add(new AdminApiServiceStep());
    result.add(new SubscriptionApiStep());
    result.add(new LeaderManagementRequestHandlerStep());

    if (config.getGateway().isEnable()) {
		// 内嵌Gateway管理
      result.add(new EmbeddedGatewayServiceStep());
    }

    // 分区管理
    result.add(new PartitionManagerStep());

    return result;
} // end buildStartupSteps
```
### (4). 设计模式
在这里的代码剖析能看出来,Zeebe运用了:装饰器模式.
### (5). 总结
通过对代码的分析,能知道BrokerStartupProcess内部主要是配置:StartupStep(这个类下一小篇进行剖析),并通过:StartupProcess Hold住所有的StartupStep. 