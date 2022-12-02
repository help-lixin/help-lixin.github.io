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
        nodes.add(node1);
        nodes.add(node2);
        nodes.add(node3);

        BootstrapService bootstrapService = bootstrapService(node1);
        NodeDiscoveryProvider nodeDiscoveryProvider = BootstrapDiscoveryProvider.builder().withNodes(nodes).build();
        nodeDiscoveryProvider.join(bootstrapService, node1).get();

        new Thread() {
            @Override
            public void run() {
                while (true) {
                    Set<Node> tmpNodes = nodeDiscoveryProvider.getNodes();
                    for (Node node : tmpNodes) {
                        System.out.println("node: " + node);
                    }
                    try {
                        TimeUnit.SECONDS.sleep(5);
                    } catch (Exception e) {
                    }
                }
            }
        }.start();

        TimeUnit.SECONDS.sleep(120);
    }

    public BootstrapService bootstrapService(Node node) {
        String cluster = "default";
        MessagingConfig messagingConfig = new MessagingConfig();
        ManagedMessagingService messagingService = new NettyMessagingService(cluster, node.address(), messagingConfig);
        messagingService.start();

        BroadcastService broadcastService = new NettyBroadcastService(node.address(), Address.from("231.0.0.1:8888"), true);
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
        nodes.add(node2);
        nodes.add(node3);

        BootstrapService bootstrapService = bootstrapService(node2);
        NodeDiscoveryProvider nodeDiscoveryProvider = BootstrapDiscoveryProvider.builder().withNodes(nodes).build();
        nodeDiscoveryProvider.join(bootstrapService, node2).get();

        new Thread() {
            @Override
            public void run() {
                while (true) {
                    Set<Node> tmpNodes = nodeDiscoveryProvider.getNodes();
                    for (Node node : tmpNodes) {
                        System.out.println("node: " + node);
                    }
                    try {
                        TimeUnit.SECONDS.sleep(5);
                    } catch (Exception e) {
                    }
                }
            }
        }.start();

        TimeUnit.SECONDS.sleep(120);
    }

    public BootstrapService bootstrapService(Node node) {
        String cluster = "default";
        MessagingConfig messagingConfig = new MessagingConfig();
        ManagedMessagingService messagingService = new NettyMessagingService(cluster, node.address(), messagingConfig);
        messagingService.start();

        BroadcastService broadcastService = new NettyBroadcastService(node.address(), Address.from("231.0.0.1:8888"), true);

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
import io.atomix.cluster.discovery.BootstrapDiscoveryProvider;
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
        nodes.add(node2);
        nodes.add(node3);

        BootstrapService bootstrapService = bootstrapService(node2);
        NodeDiscoveryProvider nodeDiscoveryProvider = BootstrapDiscoveryProvider.builder().withNodes(nodes).build();
        nodeDiscoveryProvider.join(bootstrapService, node2).get();

        new Thread() {
            @Override
            public void run() {
                while (true) {
                    Set<Node> tmpNodes = nodeDiscoveryProvider.getNodes();
                    for (Node node : tmpNodes) {
                        System.out.println("node: " + node);
                    }
                    try {
                        TimeUnit.SECONDS.sleep(5);
                    } catch (Exception e) {
                    }
                }
            }
        }.start();

        TimeUnit.SECONDS.sleep(120);
    }

    public BootstrapService bootstrapService(Node node) {
        String cluster = "default";
        MessagingConfig messagingConfig = new MessagingConfig();
        ManagedMessagingService messagingService = new NettyMessagingService(cluster, node.address(), messagingConfig);
        messagingService.start();

        BroadcastService broadcastService = new NettyBroadcastService(node.address(), Address.from("231.0.0.1:8888"), true);

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
这个案例的目的在于,能及时知道其它节点的上线和离线.