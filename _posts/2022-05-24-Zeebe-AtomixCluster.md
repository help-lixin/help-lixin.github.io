---
layout: post
title: 'Zeebe源码之AtomixCluster(五)' 
date: 2022-05-24
author: 李新
tags:  Zeebe
---

### (1). 概述
AtomixCluster是由AtomixClusterFactory工厂创建,所以,要先看下:AtomixCluster的构建过程.
### (2). AtomixClusterFactory
```
public final class AtomixClusterFactory {

  private static final Logger LOG = Loggers.CLUSTERING_LOGGER;

  private AtomixClusterFactory() {}

  // 读取brokder配置,创建:AtomixCluster对象
  public static AtomixCluster fromConfiguration(final BrokerCfg configuration) {
    final var clusterCfg = configuration.getCluster();
    final var nodeId = clusterCfg.getNodeId();
    final var localMemberId = Integer.toString(nodeId);
    final var networkCfg = configuration.getNetwork();

    final var discoveryProvider = createDiscoveryProvider(clusterCfg, localMemberId);

    final var membershipCfg = clusterCfg.getMembership();
    final var membershipProtocol =
        SwimMembershipProtocol.builder()
            .withFailureTimeout(membershipCfg.getFailureTimeout())
            .withGossipInterval(membershipCfg.getGossipInterval())
            .withProbeInterval(membershipCfg.getProbeInterval())
            .withProbeTimeout(membershipCfg.getProbeTimeout())
            .withBroadcastDisputes(membershipCfg.isBroadcastDisputes())
            .withBroadcastUpdates(membershipCfg.isBroadcastUpdates())
            .withGossipFanout(membershipCfg.getGossipFanout())
            .withNotifySuspect(membershipCfg.isNotifySuspect())
            .withSuspectProbes(membershipCfg.getSuspectProbes())
            .withSyncInterval(membershipCfg.getSyncInterval())
            .build();

    final var atomixBuilder =
        new AtomixClusterBuilder(new ClusterConfig())
		     //zeebe-cluster
            .withClusterId(clusterCfg.getClusterName())
			// 0 
            .withMemberId(localMemberId)
            .withMembershipProtocol(membershipProtocol)
			// 0.0.0.0
            .withMessagingInterface(networkCfg.getInternalApi().getHost())
			// 26502
            .withMessagingPort(networkCfg.getInternalApi().getPort())
            .withAddress(
                Address.from(
				    // 0.0.0.0
                    networkCfg.getInternalApi().getAdvertisedHost(),
					// 26502
                    networkCfg.getInternalApi().getAdvertisedPort()))
            .withMembershipProvider(discoveryProvider)
			// NONE
            .withMessageCompression(clusterCfg.getMessageCompression());

    final var securityCfg = networkCfg.getSecurity();
    if (securityCfg.isEnabled()) { 
      atomixBuilder.withSecurity(securityCfg.getCertificateChainPath(), securityCfg.getPrivateKeyPath());
    }
	
	// ********************************************************************
	// 通过AtomixClusterBuilder构建出:AtomixCluster
	// ********************************************************************
    return atomixBuilder.build();
  }

  private static NodeDiscoveryProvider createDiscoveryProvider(
      final ClusterCfg clusterCfg, final String localMemberId) {
    final BootstrapDiscoveryBuilder builder = BootstrapDiscoveryProvider.builder();
    final List<String> initialContactPoints = clusterCfg.getInitialContactPoints();

    final List<Node> nodes = new ArrayList<>();
    initialContactPoints.forEach(
        contactAddress -> {
          final Node node = Node.builder().withAddress(Address.from(contactAddress)).build();
          LOG.debug("Member {} will contact node: {}", localMemberId, node.address());
          nodes.add(node);
        });
    return builder.withNodes(nodes).build();
  }
}
```
### (3). AtomixCluster构建器
```
public AtomixCluster(final ClusterConfig config, final Version version) {
	this(config, version, buildMessagingService(config), buildUnicastService(config));
}
```
### (4). AtomixCluster.buildMessagingService
```
protected static ManagedMessagingService buildMessagingService(final ClusterConfig config) {
	// *************************************************************
	// config.getNodeConfig().getAddress()  = 0.0.0.0:26502
	// NettyMessagingService主要用于TCP通信.
	// *************************************************************
	return new NettyMessagingService(config.getClusterId(), config.getNodeConfig().getAddress(), config.getMessagingConfig());
}
```
### (5).  AtomixCluster.buildUnicastService
```
protected static ManagedUnicastService buildUnicastService(final ClusterConfig config) {
	// *************************************************************
	// NettyUnicastService广播通信.
	// *************************************************************
    return new NettyUnicastService(config.getClusterId(), config.getNodeConfig().getAddress(), config.getMessagingConfig());
}
```
### (6). 
```
protected AtomixCluster(
      final ClusterConfig config,
      final Version version,
      final ManagedMessagingService messagingService,
      final ManagedUnicastService unicastService) {
    this.messagingService =
        messagingService != null ? messagingService : buildMessagingService(config);
    this.unicastService = unicastService != null ? unicastService : buildUnicastService(config);

    discoveryProvider = buildLocationProvider(config);
    membershipProtocol = buildMembershipProtocol(config);
	
	// ************************************************************************
	// 构建集群成员服务
	// ************************************************************************
    membershipService = buildClusterMembershipService(config, this, discoveryProvider, membershipProtocol, version);
	
	// ************************************************************************
	// 构建管理集群通信的服务
	// ************************************************************************
    communicationService = buildClusterMessagingService(getMembershipService(), getMessagingService(), getUnicastService());
	
	// ************************************************************************
	// 构建集群事件通信服务
	// ************************************************************************
    eventService = buildClusterEventService(getMembershipService(), getMessagingService());
}
```
### (7). AtomixCluster.buildClusterMessagingService
```
protected static ManagedClusterCommunicationService buildClusterMessagingService(
      final ClusterMembershipService membershipService,
      final MessagingService messagingService,
      final UnicastService unicastService) {
	return new DefaultClusterCommunicationService(membershipService, messagingService, unicastService);
}
```
### (8). AtomixCluster.buildClusterEventService
```
protected static ManagedClusterEventService buildClusterEventService(
      final ClusterMembershipService membershipService, final MessagingService messagingService) {
	return new DefaultClusterEventService(membershipService, messagingService);
}
```
### (9). 总结s