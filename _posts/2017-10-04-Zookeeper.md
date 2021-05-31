---
layout: post
title: '基于Zookeeper实现服务注册功能'
date: 2017-10-04
author: 李新
tags: Zookeeper
---

### (1). 前言
> 有这样一个需求:在IM服务器启动时,会把自己的ip的port向ZK进行注册,这个port主要进行消息路由功能(不对外服务).  
> 所有的IM会获取这个IM服务列表,与这些IM服务列表保持着长连接.

### (2). 实体定义
```
package help.lixin.zk.entity;

import java.util.Objects;

public class NodeInfo {
    private String ip;
    private String port;

    public String getIp() {
        return ip;
    }

    public void setIp(String ip) {
        this.ip = ip;
    }

    public String getPort() {
        return port;
    }

    public void setPort(String port) {
        this.port = port;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        NodeInfo nodeInfo = (NodeInfo) o;
        return Objects.equals(ip, nodeInfo.ip) && Objects.equals(port, nodeInfo.port);
    }

    @Override
    public int hashCode() {
        return Objects.hash(ip, port);
    }

    @Override
    public String toString() {
        return "NodeInfo{" +
                "ip='" + ip + '\'' +
                ", port=" + port +
                '}';
    }
}
```
### (3). 接口定义
> 定义接口的目的是为了方便随意切换实现(Redis/MySQL/MongoDB).  

```
package help.lixin.zk;

import help.lixin.zk.entity.NodeInfo;

import java.util.Collection;

public interface Regsiter {

    void register() throws Exception;

    void unRegister() throws Exception;

    Collection<NodeInfo> getNodes();
}
```
### (4). 接口实现
```
package help.lixin.zk;

import help.lixin.zk.entity.NodeInfo;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.recipes.cache.PathChildrenCache;
import org.apache.curator.framework.recipes.cache.PathChildrenCacheEvent;
import org.apache.curator.framework.recipes.cache.PathChildrenCacheListener;
import org.apache.zookeeper.CreateMode;
import org.apache.zookeeper.data.Stat;

import java.nio.charset.StandardCharsets;
import java.util.Collection;
import java.util.HashSet;
import java.util.List;

public class ZKRegsiter implements Regsiter {
    private static final String PATH = "/im/nodes";
    private static final String FORMAT = "%s" + "/" + "%s";
    private CuratorFramework client = null;
    private NodeInfo currentNodeInfo;
    private String zkHosts;

    public ZKRegsiter(CuratorFramework client, NodeInfo currentNodeInfo) {
        this.currentNodeInfo = currentNodeInfo;
        this.client = client;
    }

    @Override
    public void register() throws Exception {
        String nodePath = String.format(FORMAT, PATH, currentNodeInfo.getIp());
        // 1. 创建目录
        Stat stat = client.checkExists().forPath(PATH);
        if (null == stat) {
            client.create().creatingParentsIfNeeded() // 递归
                    .withMode(CreateMode.PERSISTENT) // 持久
                    .forPath(PATH);
        }

        // 2.在目录下,创建数据
        client.create().withMode(CreateMode.EPHEMERAL) // 瞬时
                .forPath(nodePath, currentNodeInfo.getPort().getBytes(StandardCharsets.UTF_8));

        // 3. 对目录进行监听,只要目录有变化,就触发:更新
        PathChildrenCache childrenCache = new PathChildrenCache(client, PATH, true);
        childrenCache.start(PathChildrenCache.StartMode.BUILD_INITIAL_CACHE);
        childrenCache.getListenable().addListener(new PathChildrenCacheListener() {
            @Override
            public void childEvent(CuratorFramework curatorFramework, PathChildrenCacheEvent event) throws Exception {
                // TODO 设计监听器
            }
        });
    }

    @Override
    public void unRegister() throws Exception {
        String path = String.format(FORMAT, PATH, currentNodeInfo.getIp());
        Stat stat = client.checkExists().forPath(path);
        if (null != stat) {
            client.delete()
                    .guaranteed()  // 保证强制删除
                    .forPath(path);
        }
    }

    @Override
    public Collection<NodeInfo> getNodes() {
        // 重新获取ZK里的数据,并置入到缓存里
        Collection<NodeInfo> nodeInfos = new HashSet<>();
        try {
            Stat stat = client.checkExists().forPath(PATH);
            if (null != stat) {
                List<String> nodes = client.getChildren().forPath(PATH);
                for (String node : nodes) {
                    String path = String.format(FORMAT, PATH, node);
                    byte[] bytes = client.getData().forPath(path);
                    String port = new String(bytes);
                    NodeInfo nodeInfo = new NodeInfo();
                    nodeInfo.setIp(node);
                    nodeInfo.setPort(port);

                    nodeInfos.add(nodeInfo);
                }
            }
        } catch (Exception ignore) {
            // TODO WARRING
        }
        return nodeInfos;
    }
}
```
### (5). pom.xml定义
```
<!-- 对zookeeper的底层api的一些封装 -->
<dependency>
	<groupId>org.apache.curator</groupId>
	<artifactId>curator-framework</artifactId>
	<version>2.12.0</version>
</dependency>
<!-- 封装了一些高级特性，如：Cache事件监听、选举、分布式锁、分布式Barrier -->
<dependency>
	<groupId>org.apache.curator</groupId>
	<artifactId>curator-recipes</artifactId>
	<version>2.12.0</version>
</dependency>
```
### (6). 测试
```
package help.lixin;

import help.lixin.zk.Regsiter;
import help.lixin.zk.ZKRegsiter;
import help.lixin.zk.entity.NodeInfo;
import org.apache.curator.RetryPolicy;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.retry.ExponentialBackoffRetry;

import java.util.Collection;
import java.util.concurrent.TimeUnit;

public class RegsiterTest {
    public static void main(String[] args) throws  Exception {
        String zkHosts = "127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183";

        RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
        CuratorFramework client = CuratorFrameworkFactory.newClient(zkHosts,
                5000, 5000, retryPolicy);
        client.start();

        NodeInfo nodeInfo1 = new NodeInfo();
        nodeInfo1.setIp("192.168.1.80");
        nodeInfo1.setPort("9000");


        NodeInfo nodeInfo2 = new NodeInfo();
        nodeInfo2.setIp("192.168.1.90");
        nodeInfo2.setPort("9000");

		// 模拟第一个实例(JVM进程)
        Regsiter regsiter = new ZKRegsiter(client,nodeInfo1);
        regsiter.register();
        Collection<NodeInfo> nodes = regsiter.getNodes();

		// 模拟第二个实例(JVM进程)
        Regsiter regsiter2 = new ZKRegsiter(client,nodeInfo2);
        regsiter2.register();
        Collection<NodeInfo> nodes2 = regsiter2.getNodes();

        System.out.println(nodes);
        System.out.println(nodes2);
    }
}

```
### (7). 总结