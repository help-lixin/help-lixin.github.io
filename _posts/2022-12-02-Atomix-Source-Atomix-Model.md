---
layout: post
title: 'Atomix源码之业务模型(三)' 
date: 2022-12-02
author: 李新
tags:  Atomix
---

### (1). 概述
在继续往下剖析源码Atomix前,需要先看下Atomix的业务模型,即:Atomix的配置类. 

### (2). Atomix构建过程
```
Properties properties = new Properties();
Address multicastAddress = Address.from("230.0.0.1", 52307);

// 集群列表
Collection<Node> nodes = Arrays.asList(
	Node.builder().withId("1").withAddress(Address.from("localhost:5001")).build(),
	Node.builder().withId("2").withAddress(Address.from("localhost:5002")).build(),
	Node.builder().withId("3").withAddress(Address.from("localhost:5003")).build()
);

Atomix atomixClient1 = Atomix.builder()
	// MemberConfig
	.withClusterId("test")
	.withMemberId("1")
	.withAddress("localhost:5001")
	.withProperties(properties)
	
	// BootstrapDiscoveryConfig
	.withMembershipProvider(!nodes.isEmpty() ? new BootstrapDiscoveryProvider(nodes) : new MulticastDiscoveryProvider())
	
	// ProfileConfig
	.withProfiles(
			ConsensusProfile.builder()
					.withMembers("1", "2", "3")
					.withDataPath(new File(new File(DATA_DIR, "primitive-getters"), "1"))
					.build()
	)
	// MulticastConfig
	.withMulticastEnabled()
	.withMulticastAddress(multicastAddress)
.build();
```
### (3). Atomix对象依赖图
!["Atomix对象依赖图"](/assets/atomix/imgs/Atomix-Model.jpg)

### (4). 总结
从上面的代码能看出来,Atomix的构建用到了:建造者模式,并且Atomix的构建依赖很多的其它对象. 