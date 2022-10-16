---
layout: post
title: 'Zeebe ClusterServicesStep源码之ApiMessagingServiceStep(十一)' 
date: 2022-10-14
author: 李新
tags:  Zeebe
---

### (1). 概述
在这一小篇,继续把BrokerStartupProcess里的启动类剖析完.

### (2). ApiMessagingServiceStep
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
	// 0.0.0.0
    messagingConfig.setInterfaces(List.of(commandApiCfg.getHost()));
	// 26501
    messagingConfig.setPort(commandApiCfg.getPort());

    if (securityCfg.isEnabled()) {
      messagingConfig
          .setTlsEnabled(true)
          .setCertificateChain(securityCfg.getCertificateChainPath())
          .setPrivateKey(securityCfg.getPrivateKeyPath());
    }

    messagingConfig.setCompressionAlgorithm(brokerCfg.getCluster().getMessageCompression());

    final var messagingService =
        new NettyMessagingService(
            brokerCfg.getCluster().getClusterName(),
            Address.from(commandApiCfg.getAdvertisedHost(), commandApiCfg.getAdvertisedPort()),
            messagingConfig);

    messagingService
        .start()
        .whenComplete(
		// ... ... 
        );
  } // end 
}
```
### (3). 总结
Zeebe Broker在启动时,会监听26501端口,底层所使用的依然是:NettyMessagingService,后面,对于:NettyMessagingService剖析是重点.  