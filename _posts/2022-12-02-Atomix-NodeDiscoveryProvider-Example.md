---
layout: post
title: 'Atomix服务发现案例(一)' 
date: 2022-12-02
author: 李新
tags:  Atomix
---

### (1). 概述
在这一小篇,主要是把Atomix的源码,拉取下来,然后,运行一个测试案例,测试案例的需求是这样的,多个节点组成一个集群启动,能实时(上线/下线)感知其它节点. 

### (2). 节点一
```
package help.lixin.atomix.cluster;

import io.atomix.cluster.BootstrapService;
import io.atomix.cluster.Node;
import io.atomix.cluster.discovery.BootstrapDiscoveryProvider;
import io.atomix.cluster.discovery.MulticastDiscoveryBuilder;
import io.atomix.cluster.discovery.MulticastDiscoveryProvider;
import io.atomix.cluster.discovery.NodeDiscoveryProvider;
import io.atomix.cluster.messaging.BroadcastService;
import io.atomix.cluster.messaging.ManagedMessagingService;
import io.atomix.cluster.messaging.MessagingConfig;
import io.atomix.cluster.messaging.MessagingService;
import io.atomix.cluster.messaging.impl.NettyBroadcastService;
import io.atomix.cluster.messaging.impl.NettyMessagingService;
import io.atomix.utils.net.Address;
import org.junit.Test;

import java.util.ArrayList;
import java.util.List;
import java.util.Set;
import java.util.concurrent.TimeUnit;

public class NodeDiscoveryOneTest {

    @Test
    public void testNodeDiscoveryOne() throws Exception {
        List<Node> nodes = new ArrayList<Node>();

        Node node1 = Node.builder().withId("1").withAddress(Address.from("127.0.0.1", 50001)).build();
        Node node2 = Node.builder().withId("2").withAddress(Address.from("127.0.0.1", 50002)).build();
        Node node3 = Node.builder().withId("3").withAddress(Address.from("127.0.0.1", 50003)).build();
//        nodes.add(node1);
        nodes.add(node2);
        nodes.add(node3);

        BootstrapService bootstrapService = bootstrapService(node1);

        // 基于固定的节点服务发现
        NodeDiscoveryProvider nodeDiscoveryProvider = BootstrapDiscoveryProvider.builder().withNodes(nodes).build();
        nodeDiscoveryProvider.join(bootstrapService, node1).get();

        // 基于UDP服务发现
//        NodeDiscoveryProvider nodeDiscoveryProvider = MulticastDiscoveryProvider.builder().build();
        // 在此处会为:TCP/UDP进行服务发现
        // ManagedBroadcastService
        // ManagedMessagingService
//        nodeDiscoveryProvider.join(bootstrapService, node1).get();

        new Thread() {
            @Override
            public void run() {
                while (true) {
                    Set<Node> tmpNodes = nodeDiscoveryProvider.getNodes();
                    for (Node node : tmpNodes) {
                        System.out.print("node: " + node);
                    }
                    System.out.println();
                    try {
                        TimeUnit.SECONDS.sleep(5);
                    } catch (Exception e) {
                    }
                }
            }
        }.start();

        TimeUnit.SECONDS.sleep(Integer.MAX_VALUE);
    }

    public BootstrapService bootstrapService(Node node) {

        // 1.ManagedMessagingService(NettyMessagingService) 维扩的是一条TCP连接,主要用于两个Node(节点)之间的通信.
        // 2.通过MessagingService接口,可以大概猜出有如下功能: 异步发送消息/同步发送消息/注册事件处理/取消注册事件处理
        String cluster = "default";
        MessagingConfig messagingConfig = new MessagingConfig();
        ManagedMessagingService messagingService = new NettyMessagingService( //
                cluster,  // 集群名称
                node.address(), // 节点地址
                messagingConfig);
        messagingService.start();

        // 3. NettyBroadcastService用于广播.
        // 4. 会在本地创建"两个"(127.0.0.1:8888)UDP端口,并加入到广播地址(231.0.0.1:8888)中
        NettyBroadcastService broadcastService = new NettyBroadcastService( //
                node.address(), //
                Address.from("231.0.0.1:8888"), //
                true);
        broadcastService.start();

        // 4. BootstrapService包含着(ManagedMessagingService/BroadcastService),也就是广播和单播都支持
        return new BootstrapService() {
            @Override
            public MessagingService getMessagingService() {
                return messagingService;
            }

            @Override
            public BroadcastService getBroadcastService() {
                return broadcastService;
            }
        };
    }

}
```
### (3). 节点二
```
package help.lixin.atomix.cluster;

import io.atomix.cluster.BootstrapService;
import io.atomix.cluster.Node;
import io.atomix.cluster.discovery.BootstrapDiscoveryProvider;
import io.atomix.cluster.discovery.MulticastDiscoveryProvider;
import io.atomix.cluster.discovery.NodeDiscoveryProvider;
import io.atomix.cluster.messaging.BroadcastService;
import io.atomix.cluster.messaging.ManagedMessagingService;
import io.atomix.cluster.messaging.MessagingConfig;
import io.atomix.cluster.messaging.MessagingService;
import io.atomix.cluster.messaging.impl.NettyBroadcastService;
import io.atomix.cluster.messaging.impl.NettyMessagingService;
import io.atomix.utils.net.Address;
import org.junit.Test;

import java.util.ArrayList;
import java.util.List;
import java.util.Set;
import java.util.concurrent.TimeUnit;

public class NodeDiscoveryTwoTest {

    @Test
    public void testNodeDiscoveryTwo() throws Exception {
        List<Node> nodes = new ArrayList<Node>();
        Node node1 = Node.builder().withId("1").withAddress(Address.from("127.0.0.1", 50001)).build();
        Node node2 = Node.builder().withId("2").withAddress(Address.from("127.0.0.1", 50002)).build();
        Node node3 = Node.builder().withId("3").withAddress(Address.from("127.0.0.1", 50003)).build();
        nodes.add(node1);
//        nodes.add(node2);
        nodes.add(node3);

        BootstrapService bootstrapService = bootstrapService(node2);
        // 基于固定的节点服务发现
        NodeDiscoveryProvider nodeDiscoveryProvider = BootstrapDiscoveryProvider.builder().withNodes(nodes).build();
        nodeDiscoveryProvider.join(bootstrapService, node2).get();

        // 基于UDP服务发现
//        NodeDiscoveryProvider nodeDiscoveryProvider = MulticastDiscoveryProvider.builder().build();
//        nodeDiscoveryProvider.join(bootstrapService, node2).get();

        new Thread() {
            @Override
            public void run() {
                while (true) {
                    Set<Node> tmpNodes = nodeDiscoveryProvider.getNodes();
                    for (Node node : tmpNodes) {
                        System.out.print("node: " + node);
                    }
                    System.out.println();
                    try {
                        TimeUnit.SECONDS.sleep(5);
                    } catch (Exception e) {
                    }
                }
            }
        }.start();

        TimeUnit.SECONDS.sleep(Integer.MAX_VALUE);
    }

    public BootstrapService bootstrapService(Node node) {
        // 集群名称
        String cluster = "default";
        // 消息配置
        MessagingConfig messagingConfig = new MessagingConfig();
        // 当前节点(5001端口)信息
        ManagedMessagingService messagingService = new NettyMessagingService(cluster, node.address(), messagingConfig);

        messagingService.start();

        NettyBroadcastService broadcastService = new NettyBroadcastService(node.address(), Address.from("231.0.0.1:8888"), true);
        broadcastService.start();

        return new BootstrapService() {
            @Override
            public MessagingService getMessagingService() {
                return messagingService;
            }

            @Override
            public BroadcastService getBroadcastService() {
                return broadcastService;
            }
        };
    }

}
```
### (4). 节点三
```
package help.lixin.atomix.cluster;

import io.atomix.cluster.BootstrapService;
import io.atomix.cluster.Node;
import io.atomix.cluster.discovery.*;
import io.atomix.cluster.messaging.BroadcastService;
import io.atomix.cluster.messaging.ManagedMessagingService;
import io.atomix.cluster.messaging.MessagingConfig;
import io.atomix.cluster.messaging.MessagingService;
import io.atomix.cluster.messaging.impl.NettyBroadcastService;
import io.atomix.cluster.messaging.impl.NettyMessagingService;
import io.atomix.utils.net.Address;
import org.junit.Test;

import java.util.ArrayList;
import java.util.List;
import java.util.Set;
import java.util.concurrent.TimeUnit;

public class NodeDiscoveryThreeTest {

    @Test
    public void testNodeDiscoveryThree() throws Exception {
        List<Node> nodes = new ArrayList<Node>();
        Node node1 = Node.builder().withId("1").withAddress(Address.from("127.0.0.1", 50001)).build();
        Node node2 = Node.builder().withId("2").withAddress(Address.from("127.0.0.1", 50002)).build();
        Node node3 = Node.builder().withId("3").withAddress(Address.from("127.0.0.1", 50003)).build();
        nodes.add(node1);
        nodes.add(node2);
//        nodes.add(node3);

        BootstrapService bootstrapService = bootstrapService(node3);
        // 基于固定的节点服务发现
        NodeDiscoveryProvider nodeDiscoveryProvider = BootstrapDiscoveryProvider.builder().withNodes(nodes).build();
        nodeDiscoveryProvider.join(bootstrapService, node3).get();

        // 基于UDP服务发现
//        NodeDiscoveryProvider nodeDiscoveryProvider = MulticastDiscoveryProvider.builder().build();
//        nodeDiscoveryProvider.join(bootstrapService, node3).get();

        new Thread() {
            @Override
            public void run() {
                while (true) {
                    Set<Node> tmpNodes = nodeDiscoveryProvider.getNodes();
                    for (Node node : tmpNodes) {
                        System.out.print("node: " + node);
                    }
                    System.out.println();
                    try {
                        TimeUnit.SECONDS.sleep(5);
                    } catch (Exception e) {
                    }
                }
            }
        }.start();

        TimeUnit.SECONDS.sleep(Integer.MAX_VALUE);
    }

    public BootstrapService bootstrapService(Node node) {
        String cluster = "default";
        MessagingConfig messagingConfig = new MessagingConfig();
        ManagedMessagingService messagingService = new NettyMessagingService(cluster, node.address(), messagingConfig);
        messagingService.start();

        NettyBroadcastService broadcastService = new NettyBroadcastService(node.address(), Address.from("231.0.0.1:8888"), true);
        broadcastService.start();

        return new BootstrapService() {
            @Override
            public MessagingService getMessagingService() {
                return messagingService;
            }

            @Override
            public BroadcastService getBroadcastService() {
                return broadcastService;
            }
        };
    }

}
```
### (5). 输出结果
```
node: Node{id=1, address=127.0.0.1:50001}
node: Node{id=2, address=127.0.0.1:50002}
node: Node{id=3, address=127.0.0.1:50003}
```
### (6). 总结
这个案例的目的在于,能及时知道其它节点的上线和离线(而且支持TCP/UDP的方式,可以自由切换).