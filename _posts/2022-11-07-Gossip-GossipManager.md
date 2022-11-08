---
layout: post
title: 'Gossip源码之GossipManager(三)' 
date: 2022-11-07
author: 李新
tags:  Gossip
---


### (1). 概述

### (2). GossipManager构建过程
> 需要注意uri所指向的是本机信息,而,gossipMembers所指向的是其它成员信息

```
GossipSettings s = new GossipSettings();
s.setWindowSize(1000);
s.setGossipInterval(100);

// GossipManager通过Builder模式创建
GossipManager gossipService = GossipManagerBuilder.newBuilder()
        .cluster("mycluster")
		// 本机成员信息,以及唯一id
		// udp://localhost:10000 0
		.uri(URI.create("udp://localhost:10000")).id("0")
		// 其它成员信息,以及,其它成员的唯一id
		// udp://localhost:10001 1
		.gossipMembers(Collections.singletonList(new RemoteMember("mycluster", URI.create("udp://localhost:10001"), "1")))
		.gossipSettings(s)
		.build();
```
### (3). GossipManager构建器
```
// ********************************************************************
// 所有的成员列表信息,但是,不包含当前节点(自己).
// ********************************************************************
private final ConcurrentSkipListMap<LocalMember, GossipState> members;

// 当前节点信息
private final LocalMember me;


public GossipManager(String cluster,
                         URI uri, String id, Map<String, String> properties, GossipSettings settings,
                         List<Member> gossipMembers, GossipListener listener, MetricRegistry registry,
                         MessageHandler messageHandler) {
    // ... ...
								 
	// *********************************************************************************
	// 根据uri(udp://localhost:10000),创建:LocalMember
	// *********************************************************************************
	me = new LocalMember(cluster, uri, id, clock.nanoTime(), properties, settings.getWindowSize(), settings.getMinimumSamples(), settings.getDistribution());
	

	// *********************************************************************************
	// gossipMembers = [ udp://localhost:10001 ]
	// 把种子节点(udp://localhost:10001),转换成:LocalMember,并设置状态为下线(DOWN)
	// *********************************************************************************
	members = new ConcurrentSkipListMap<>();
	for (Member startupMember : gossipMembers) {
		if (!startupMember.equals(me)) {
			LocalMember member = new LocalMember(startupMember.getClusterName(),
					startupMember.getUri(), startupMember.getId(),
					clock.nanoTime(), startupMember.getProperties(), settings.getWindowSize(),
					settings.getMinimumSamples(), settings.getDistribution());
			//TODO should members start in down state?
			// 
			members.put(member, GossipState.DOWN);
		}
	}
	
    // ... ...
} // end 
```
### (4). GossipManager.init
```
public void init() {
	// 通过反射创建:协议处理(实际就是读写报文)
	// org.apache.gossip.protocol.json.JacksonProtocolManager
	protocolManager = ReflectionUtils.constructWithReflection(
			settings.getProtocolManagerClass(),
			new Class<?>[]{GossipSettings.class, String.class, MetricRegistry.class},
			new Object[]{settings, me.getId(), this.getRegistry()}
	);

	// *************************************************************
	// org.apache.gossip.transport.udp.UdpTransportManager
	// *************************************************************
	// 通过反射创建:传输层管理
	transportManager = ReflectionUtils.constructWithReflection(
			settings.getTransportManagerClass(),
			new Class<?>[]{GossipManager.class, GossipCore.class},
			new Object[]{this, gossipCore}
	);

	// *************************************************************
	// startEndpoint用于监听端口,接受请求并处理.
	// startActiveGossiper用于定时随机向成员发送成员列表数据.
	// *************************************************************
	// start processing gossip messages.
	transportManager.startEndpoint();
	
	// startActiveGossiper
	transportManager.startActiveGossiper();
	// ... ...
} // end init 
```
### (5). 总结
GossipManager的内部有两个成员属性比较重要,一个是:me,另一个是:members,在GossipManager初始化时,传入的种子member(成员),默认状态就是下线(DOWN),在调用init方法时,会通过反射创建:TransportManager实例,进行消息的同步,所以,下一小篇开始会着重分析:UdpTransportManager.  