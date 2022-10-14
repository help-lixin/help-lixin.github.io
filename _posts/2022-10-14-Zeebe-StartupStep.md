---
layout: post
title: 'Zeebe源码之StartupStep(三)' 
date: 2022-10-14
author: 李新
tags:  Zeebe
---

### (1). 概述
在前面,剖析到StartupProcess会持有一堆StartupStep,那么,StartupStep是什么呢?它的作用是什么呢?在这一小篇主要对StartupStep(ApiMessagingServiceStep)进行剖析.  
### (2). 查看ApiMessagingServiceStep的继承关系
```
io.camunda.zeebe.util.startup.StartupStep
    io.camunda.zeebe.broker.bootstrap.AbstractBrokerStartupStep
	    io.camunda.zeebe.broker.bootstrap.ApiMessagingServiceStep
```
### (3). 接口StartupStep
> 看接口,大概能知道这个接口,主要是负责启动和关闭,而,StartupProcess负责调度所有的:StartupStep.  

```
public interface StartupStep<CONTEXT> {
	String getName();
	
	// 启动
	ActorFuture<CONTEXT> startup(final CONTEXT context);
	
	// 关闭
	ActorFuture<CONTEXT> shutdown(final CONTEXT context);
}	
```
### (4). 抽象类(AbstractBrokerStartupStep)
```
// ***************************************************************
// AbstractBrokerStartupStep指定了具体的:CONTEXT,它的类型为:BrokerStartupContext
// 我们稍微看一下:BrokerStartupContext是什么
// ***************************************************************
abstract class AbstractBrokerStartupStep implements StartupStep<BrokerStartupContext> {
	
	// 预留两个方法,给子类(比如:ApiMessagingServiceStep)去实现. 
	// ... ... 
	abstract void startupInternal(
	      final BrokerStartupContext brokerStartupContext,
	      final ConcurrencyControl concurrencyControl,
	      final ActorFuture<BrokerStartupContext> startupFuture);
	
	  abstract void shutdownInternal(
	      final BrokerStartupContext brokerShutdownContext,
	      final ConcurrencyControl concurrencyControl,
	      final ActorFuture<BrokerStartupContext> shutdownFuture);
}	
```
### (5). BrokerStartupContext
> 有一个大胆的猜测,StartupStep的实现类,实际是对:BrokerStartupContext的填充(调用相应的set方法).   

```
public interface BrokerStartupContext {

  BrokerInfo getBrokerInfo();

  BrokerCfg getBrokerConfiguration();

  SpringBrokerBridge getSpringBrokerBridge();

  ActorSchedulingService getActorSchedulingService();

  @Deprecated // use getActorSchedulingService instead
  ActorScheduler getActorScheduler();

  ConcurrencyControl getConcurrencyControl();

  BrokerHealthCheckService getHealthCheckService();

  void addPartitionListener(PartitionListener partitionListener);

  void removePartitionListener(PartitionListener partitionListener);

  List<PartitionListener> getPartitionListeners();

  ClusterServicesImpl getClusterServices();

  void setClusterServices(ClusterServicesImpl o);

  void addDiskSpaceUsageListener(DiskSpaceUsageListener listener);

  void removeDiskSpaceUsageListener(DiskSpaceUsageListener listener);

  CommandApiServiceImpl getCommandApiService();

  void setCommandApiService(CommandApiServiceImpl commandApiService);

  AdminApiRequestHandler getAdminApiService();

  void setAdminApiService(AdminApiRequestHandler adminApiService);

  AtomixServerTransport getCommandApiServerTransport();

  void setCommandApiServerTransport(AtomixServerTransport commandApiServerTransport);

  ManagedMessagingService getApiMessagingService();

  void setApiMessagingService(ManagedMessagingService commandApiMessagingService);

  SubscriptionApiCommandMessageHandlerService getSubscriptionApiService();

  void setSubscriptionApiService(
      SubscriptionApiCommandMessageHandlerService subscriptionApiService);

  EmbeddedGatewayService getEmbeddedGatewayService();

  void setEmbeddedGatewayService(EmbeddedGatewayService embeddedGatewayService);

  DiskSpaceUsageMonitor getDiskSpaceUsageMonitor();

  void setDiskSpaceUsageMonitor(DiskSpaceUsageMonitor diskSpaceUsageMonitor);

  LeaderManagementRequestHandler getLeaderManagementRequestHandler();

  void setLeaderManagementRequestHandler(final LeaderManagementRequestHandler handler);

  ExporterRepository getExporterRepository();

  PartitionManagerImpl getPartitionManager();

  void setPartitionManager(PartitionManagerImpl partitionManager);

  BrokerAdminServiceImpl getBrokerAdminService();

  void setBrokerAdminService(final BrokerAdminServiceImpl brokerAdminService);
}
```
### (6). ApiMessagingServiceStep
```
public class ApiMessagingServiceStep extends AbstractBrokerStartupStep {
  private static final Logger LOG = Loggers.SYSTEM_LOGGER;

  @Override
  void startupInternal(
      final BrokerStartupContext brokerStartupContext,
      final ConcurrencyControl concurrencyControl,
      final ActorFuture<BrokerStartupContext> startupFuture) {
    final var brokerCfg = brokerStartupContext.getBrokerConfiguration();
    final var commandApiCfg = brokerCfg.getNetwork().getCommandApi();
    final var securityCfg = brokerCfg.getNetwork().getSecurity();

    final var messagingConfig = new MessagingConfig();
    messagingConfig.setInterfaces(List.of(commandApiCfg.getHost()));
    messagingConfig.setPort(commandApiCfg.getPort());

    if (securityCfg.isEnabled()) {
      messagingConfig
          .setTlsEnabled(true)
          .setCertificateChain(securityCfg.getCertificateChainPath())
          .setPrivateKey(securityCfg.getPrivateKeyPath());
    }

    messagingConfig.setCompressionAlgorithm(brokerCfg.getCluster().getMessageCompression());
    
	// ***************************************************************
	// 暂且不看:NettyMessagingService的内容,从名称上,是不是能看出来一点什么?  
	// NettyMessagingService肯定是跟网络(会绑定某个端口或跟某个端口通信)相关了.  
	// ***************************************************************
    final var messagingService =
        new NettyMessagingService(
            brokerCfg.getCluster().getClusterName(),
			// 26501
            Address.from(commandApiCfg.getAdvertisedHost(), commandApiCfg.getAdvertisedPort()),
            messagingConfig);

    messagingService
        .start()
        .whenComplete(
            (createdMessagingService, error) -> {
              if (error != null) {
                startupFuture.completeExceptionally(error);
              } else {
                concurrencyControl.run(
                    () -> {
                      LOG.debug(
                          "Bound API to {}, using advertised address {} ",
                          messagingService.bindingAddresses(),
                          messagingService.address());
                      brokerStartupContext.setApiMessagingService(messagingService);
                      startupFuture.complete(brokerStartupContext);
                    });
              }
            });
  } // end startupInternal
}
```
### (7). 总结
分析了半天,然后,StartupStep是一个生命周期管理系统,它最终的目标是填充BrokerStartupContext里所有的set方法. 