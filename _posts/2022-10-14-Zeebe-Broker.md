---
layout: post
title: 'Zeebe源码之Broker初始化(一)' 
date: 2022-10-14
author: 李新
tags:  Zeebe
---

### (1). 概述
在前面主要剖析Zeebe源码的组件之一的:Gateway,从这里开始,将对Broker进行剖析.
### (2). Broker构造器
```
public final class Broker implements AutoCloseable {

  public static final Logger LOG = Loggers.SYSTEM_LOGGER;

  private final SystemContext systemContext;
  private boolean isClosed = false;

  // 
  private CompletableFuture<Broker> startFuture;
  private final ActorScheduler scheduler;
  
  // 健康心跳监测
  private BrokerHealthCheckService healthCheckService;
  // 
  private final BrokerStartupActor brokerStartupActor;
  // 
  private final BrokerInfo localBroker;
  
  
  private BrokerContext brokerContext;
  
  
  public Broker(
    // ***********************************************************
    // SystemContext的目的是对:配置信息和Scheduler. 
    // ***********************************************************
	final SystemContext systemContext,
	
	// ***********************************************************
	// SpringBrokerBridge相当于一个中介者模式,当Broker全部初始化完成之后,会把一些信息Service注册到SpringBrokerBridge里,
	// 然后,其它类,可以从SpringBrokerBridge里获取.
	// ***********************************************************
	final SpringBrokerBridge springBrokerBridge,
	
	// ***********************************************************
	// 分区监听器,主要用于当分区为Leader,或者,分区状态完成为活动状态时的回调. 
	// ***********************************************************
	final List<PartitionListener> additionalPartitionListeners) {
      // 系统上下文
	  this.systemContext = systemContext;
	  // Scheduler为调度器
	  scheduler = this.systemContext.getScheduler();
	  // 创建:BrokerInfo
	  localBroker = createBrokerInfo(getConfig());
	  healthCheckService = new BrokerHealthCheckService(localBroker);

	  final BrokerStartupContextImpl startupContext =
		  new BrokerStartupContextImpl(
			  localBroker,
			  systemContext.getBrokerConfiguration(),
			  springBrokerBridge,
			  scheduler,
			  healthCheckService,
			  // *************************************************************
			  // 1. 通过配置文件,加载Exporter
			  // *************************************************************
			  buildExporterRepository(getConfig()),
			  additionalPartitionListeners);

      // *************************************************************
	  // 2. 创建BrokerStartupActor专门用来管理:Broker的启动
	  // *************************************************************
	  brokerStartupActor = new BrokerStartupActor(startupContext);
	  scheduler.submitActor(brokerStartupActor);
	} // end
	
	// ***********************************************************************
	// 通过配置文件,加载Exporter
	// ***********************************************************************
	private ExporterRepository buildExporterRepository(final BrokerCfg cfg) {
	    final ExporterRepository exporterRepository = new ExporterRepository();
	    final var exporterEntries = cfg.getExporters().entrySet();
		
	    for (final var exporterEntry : exporterEntries) {
	      final var id = exporterEntry.getKey();
	      final var exporterCfg = exporterEntry.getValue();
	      try {
	        exporterRepository.load(id, exporterCfg);
	      } catch (final ExporterLoadException | ExternalJarLoadException e) {
	        throw new IllegalStateException("Failed to load exporter with configuration: " + exporterCfg, e);
	      }
	    }// end for
	    return exporterRepository;
	} //end buildExporterRepository
}  
```
### (3). BrokerStartupActor构建器
```
private BrokerStartupActor(final BrokerStartupContextImpl startupContext) {
	nodeId = startupContext.getBrokerInfo().getNodeId();
	startupContext.setConcurrencyControl(actor);
	// *******************************************************
	// 3. 创建:BrokerStartupProcess,该类主要负责Broker启动的相关管理.
	//    下一小节,专门分析:BrokerStartupProcess,
	// *******************************************************
	brokerStartupProcess = new BrokerStartupProcess(startupContext);
}
```
### (4). 总结
Broker构造器主要做以下几件事:    
+ 创建BrokerInfo,它主要是Broker基本信息的展示.   
+ 创建BrokerHealthCheckService,一看类名称就知道是跟Health相关.  
+ 创建ExporterRepository,它主要负责加载:导出器(Exporter).  
+ 创建BrokerStartupContext,它仅仅是启动时的一个上下文,承载着启动时需要的一些信息.   
+ 创建BrokerStartupActor,它主要负责Broker启动处理,内部持有一个:BrokerStartupProcess对象,BrokerStartupProcess的内容,下一小篇进行部析.  