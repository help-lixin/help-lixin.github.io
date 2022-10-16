---
layout: post
title: 'Zeebe ClusterServicesStep源码之ClusterEventService(十)' 
date: 2022-10-14
author: 李新
tags:  Zeebe
---

### (1). 概述

### (2). AtomixCluster.buildClusterEventService
```
protected static ManagedClusterEventService buildClusterEventService(
      final ClusterMembershipService membershipService, 
	  final MessagingService messagingService) {
    return new DefaultClusterEventService(membershipService, messagingService);
}
```
### (3). 看下DefaultClusterEventService类的继承关系
```
io.atomix.cluster.messaging.ClusterEventService
  io.atomix.cluster.messaging.ManagedClusterEventService
    io.atomix.cluster.messaging.impl.DefaultClusterEventService
```
### (4). ClusterEventService
```
public interface ClusterEventService {
	
	<M> void broadcast(String topic, M message, Function<M, byte[]> encoder);
	
	<M, R> CompletableFuture<Subscription> subscribe(String topic,Function<byte[], M> decoder,Function<M, R> handler,Function<R, byte[]> encoder,Executor executor);
		  	  
	<M, R> CompletableFuture<Subscription> subscribe(String topic,Function<byte[], M> decoder,Function<M, CompletableFuture<R>> handler,Function<R, byte[]> encoder);	  
	
	<M> CompletableFuture<Subscription> subscribe(String topic, Function<byte[], M> decoder, Consumer<M> handler, Executor executor);
	
	List<Subscription> getSubscriptions(String topic);
	
	Set<MemberId> getSubscribers(String topic);
}	
```
### (5). 总结
ClusterEventService的主要职责是根据topic广播事件,以及根据topic订阅事件.  