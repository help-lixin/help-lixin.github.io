---
layout: post
title: 'Atomix分布式基本数据类型操作(二)' 
date: 2022-12-02
author: 李新
tags:  Atomix
---

### (1). 概述
在这一小篇,主要学习Atomix在分布式环境下,基本数据类型的使用.  

### (2). 案例代码
```
package io.atomix.core.multimap;

import io.atomix.cluster.Member;
import io.atomix.cluster.Node;
import io.atomix.cluster.discovery.BootstrapDiscoveryProvider;
import io.atomix.cluster.discovery.MulticastDiscoveryProvider;
import io.atomix.core.Atomix;
import io.atomix.core.lock.DistributedLock;
import io.atomix.core.map.AtomicMap;
import io.atomix.core.profile.ConsensusProfile;
import io.atomix.core.profile.Profile;
import io.atomix.primitive.protocol.ProxyProtocol;
import io.atomix.protocols.raft.MultiRaftProtocol;
import io.atomix.utils.net.Address;

import java.io.File;
import java.util.Arrays;
import java.util.Collection;
import java.util.Properties;
import java.util.Set;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;
import java.util.stream.Stream;

public class RaftClientTest {
    protected static final File DATA_DIR = new File(System.getProperty("user.dir"), ".data");

    public static void main(String[] args) throws Exception {
        Properties properties = new Properties();
        Address multicastAddress = Address.from("230.0.0.1", 52307);

        // 集群列表
        Collection<Node> nodes = Arrays.asList(
                Node.builder().withId("1").withAddress(Address.from("localhost:5001")).build(),
                Node.builder().withId("2").withAddress(Address.from("localhost:5002")).build(),
                Node.builder().withId("3").withAddress(Address.from("localhost:5003")).build());

        Atomix atomixClient1 = Atomix.builder()
                .withClusterId("test")
                .withMemberId("1")
                .withAddress("localhost:5001")
                .withProperties(properties)
                .withMembershipProvider(!nodes.isEmpty() ? new BootstrapDiscoveryProvider(nodes) : new MulticastDiscoveryProvider())
                .withProfiles(
                        ConsensusProfile.builder()
                                .withMembers("1", "2", "3")
                                .withDataPath(new File(new File(DATA_DIR, "primitive-getters"), "1"))
                                .build()
                )
                .withMulticastEnabled()
                .withMulticastAddress(multicastAddress)
                .build();
				
        Atomix atomixClient2 = Atomix.builder()
                .withClusterId("test")
                .withMemberId("2")
                .withAddress("localhost:5002")
                .withProperties(properties)
                .withMembershipProvider(!nodes.isEmpty() ? new BootstrapDiscoveryProvider(nodes) : new MulticastDiscoveryProvider())
                .withProfiles(
                        ConsensusProfile.builder()
                                .withMembers("1", "2", "3")
                                .withDataPath(new File(new File(DATA_DIR, "primitive-getters"), "2"))
                                .build()
                )
                .withMulticastEnabled()
                .withMulticastAddress(multicastAddress)
                .build();
				
        Atomix atomixClient3 = Atomix.builder()
                .withClusterId("test")
                .withMemberId("3")
                .withAddress("localhost:5003")
                .withProperties(properties)
                .withMembershipProvider(!nodes.isEmpty() ? new BootstrapDiscoveryProvider(nodes) : new MulticastDiscoveryProvider())
                .withProfiles( // 一致性配置
                        ConsensusProfile.builder()
                                .withMembers("1", "2", "3")
                                .withDataPath(new File(new File(DATA_DIR, "primitive-getters"), "3"))
                                .build()
                )
                .withMulticastEnabled()
                .withMulticastAddress(multicastAddress)
                .build();

        atomixClient1.start();
        atomixClient2.start();
        atomixClient3.start();

        TimeUnit.SECONDS.sleep(30);

        // 创建一个客户端(不存储数据)
        Atomix atomixClient = Atomix.builder()
                .withClusterId("test")
                .withMemberId("4")
                .withAddress("localhost:5004")
                .withProperties(properties)
                .withMembershipProvider(!nodes.isEmpty() ? new BootstrapDiscoveryProvider(nodes) : new MulticastDiscoveryProvider())
                .withProfiles(
                        Profile.client()
                )
                .withMulticastEnabled()
                .withMulticastAddress(multicastAddress)
                .build();
        atomixClient.start().get(30, TimeUnit.SECONDS);

        System.out.println("member: " + atomixClient.getMembershipService().getMembers().size());
         
		// 分布式map操作 
        AtomicMap<String, String> testMap = atomixClient.<String, String>getAtomicMap("test");
        testMap.put("hello", "world");

        TimeUnit.SECONDS.sleep(10);
		// 分布式map读取
        AtomicMap<String, String> atomicMap = atomixClient1.<String, String>getAtomicMap("test");
        System.out.println("map value:" + atomicMap.get("hello")); 
    }
}
```
### (3). 输出结果
```
member: 4
map value:Versioned{value=world, version=5, creationTime=2022-12-03 04:03:16,930}
```
### (4). 总结
