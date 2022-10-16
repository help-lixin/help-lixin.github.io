---
layout: post
title: 'Zeebe ClusterServicesStep源码之ClusterCommunicationService(九)' 
date: 2022-10-14
author: 李新
tags:  Zeebe
---

### (1). 概述
在这一小篇,主要剖析:ClusterCommunicationService,通过类的命名,大概就能看出来,它主要负责:集群通信.  

### (2). AtomixCluster.buildClusterMessagingService
```
 protected static ManagedClusterCommunicationService buildClusterMessagingService(
      // DefaultClusterMembershipService负责通过TCP/UDP方式同步集群成员列表
      final ClusterMembershipService membershipService,
	  // NettyMessagingService(是一个专门用来通信的工具)
      final MessagingService messagingService,
	  // 看名字就知道了,是用于广播用途
      final UnicastService unicastService) {
    return new DefaultClusterCommunicationService(membershipService, messagingService, unicastService);
}
```
### (3). 查看DefaultClusterCommunicationService类的继承关系
```
io.atomix.cluster.messaging.ClusterCommunicationService
    io.atomix.cluster.messaging.ManagedClusterCommunicationService
	  io.atomix.cluster.messaging.impl.DefaultClusterCommunicationService
```

### (4). 看下ClusterCommunicationService接口的能力
```
public interface ClusterCommunicationService {
	
	<M> void broadcast(String subject, M message, Function<M, byte[]> encoder, boolean reliable);
	
	<M> void multicast(String subject ,M message ,Function<M, byte[]> encoder ,Set<MemberId> memberIds, boolean reliable);
	
	<M> void unicast(String subject, M message, Function<M, byte[]> encoder, MemberId memberId, boolean reliable);
	
	<M, R> CompletableFuture<R> send(String subject, M message, Function<M, byte[]> encoder, Function<byte[], R> decoder, MemberId toMemberId, Duration timeout);
	
	<M, R> CompletableFuture<Void> subscribe( String subject, Function<byte[], M> decoder, Function<M, R> handler, Function<R, byte[]> encoder, Executor executor);
	
	<M, R> CompletableFuture<Void> subscribe( String subject, Function<byte[], M> decoder, Function<M, CompletableFuture<R>> handler, Function<R, byte[]> encoder);
    
	<M> CompletableFuture<Void> subscribe(String subject, Function<byte[], M> decoder, Consumer<M> handler, Executor executor);
	
	<M> CompletableFuture<Void> subscribe( String subject, Function<byte[], M> decoder, BiConsumer<MemberId, M> handler, Executor executor);
	
	void unsubscribe(String subject);
}	
```
### (5). 总结
实现类就不看了,ClusterCommunicationService接口的能力,通过方法签名就能看出来,主要用于消息通信来着.  