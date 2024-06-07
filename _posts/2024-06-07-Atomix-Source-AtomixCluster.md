---
layout: post
title: 'Atomix之AtomixCluster入门案例' 
date: 2024-06-07
author: 李新
tags:  Atomix
---

### (1). 概述

以前为了图方便换成看JRaft源码,最近还是想把:Atomix的源码继续看完,所以,先以一些简单的Demo入门,然后,才进行深入的剖析.

### (2). AtomixClusterServer案例
```
package help.lixin.atomix.cluster;

import io.atomix.cluster.*;
import io.atomix.cluster.discovery.BootstrapDiscoveryProvider;
import io.atomix.cluster.messaging.ClusterCommunicationService;
import io.atomix.utils.net.Address;

import java.util.Arrays;
import java.util.Collection;
import java.util.Set;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class AtomixClusterServer {
    public static void main(String[] args) throws InterruptedException {
        Collection<Node> bootstrapLocations = Arrays.asList(
                // 这里的foo为节点的唯一标识
                Node.builder().withId("foo").withAddress(Address.from("localhost:5000")).build(),
                Node.builder().withId("bar").withAddress(Address.from("localhost:5001")).build(),
                Node.builder().withId("baz").withAddress(Address.from("localhost:5002")).build());

        AtomixCluster cluster1 = AtomixCluster.builder()
                .withClusterId("test")
                // localMemberId 为本地成员标识
                .withMemberId("foo")
                // 要绑定地址
                .withAddress("localhost:5000")
                // 发现其它成员的提供者
                .withMembershipProvider(BootstrapDiscoveryProvider.builder()
                        .withNodes(bootstrapLocations)
                        .build())
                .build();
        cluster1.start().join();


        AtomixCluster cluster2 = AtomixCluster.builder()
                .withClusterId("test")
                .withMemberId("bar")
                .withAddress("localhost:5001")
                .withMembershipProvider(BootstrapDiscoveryProvider.builder()
                        .withNodes(bootstrapLocations)
                        .build())
                .build();
        cluster2.start().join();

        cluster2.getCommunicationService().subscribe("hello", message -> {
            System.out.println("member[bar] revice message:" + message);
        }, Executors.newSingleThreadExecutor());

        AtomixCluster cluster3 = AtomixCluster.builder()
                .withClusterId("test")
                .withMemberId("baz")
                .withAddress("localhost:5002")
                .withMembershipProvider(BootstrapDiscoveryProvider.builder()
                        .withNodes(bootstrapLocations)
                        .build())
                .build();
        cluster3.start().join();

        new Thread() {
            @Override
            public void run() {
                while (true) {
                    try {
                        TimeUnit.SECONDS.sleep(30);
                    } catch (Exception e) {
                    }
                    System.out.println("****************************************************************");
                    ClusterMembershipService membershipService = cluster3.getMembershipService();
                    Set<Member> members = membershipService.getReachableMembers();
                    for (Member member : members) {
                        System.out.print(member);
                    }
                    System.out.println("\n****************************************************************");
                }
            }
        }.start();


        new Thread() {
            @Override
            public void run() {
                try {
                    ClusterCommunicationService communicationService = cluster3.getCommunicationService();
                    MemberId bar = MemberId.from("bar");
                    communicationService.send("hello", "world!", bar);
                    TimeUnit.SECONDS.sleep(10);
                    communicationService.send("hello", "world!!", bar);
                    TimeUnit.SECONDS.sleep(10);
                    communicationService.send("hello", "world!!!", bar);
                } catch (Exception ignore) {
                }
            }
        }.start();

        TimeUnit.SECONDS.sleep(1000000);
    }
}
```
### (3). 总结
通过手工创建三个Member进行,可以实现节点间的服务发现功能. 
