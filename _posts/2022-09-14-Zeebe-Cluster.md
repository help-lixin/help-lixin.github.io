---
layout: post
title: 'Zeebe集群搭建' 
date: 2022-09-14
author: 李新
tags:  Zeebe
---

### (1).  机器准备

|  机器名称   | 机器IP         |
|  ----      | ----          |
| gateway    | 10.211.55.100 |
| node-1     | 10.211.55.101 |
| node-2     | 10.211.55.102 |
| node-3     | 10.211.55.103 |

### (2). 配置hosts
```
[root@gateway bin]# cat /etc/hosts
10.211.55.100 gateway
10.211.55.101 node-1
10.211.55.102 node-2
10.211.55.103 node-3
```
### (3). 源码下载并编译
```
# 我这里以1.4为案例.
/Users/lixin/GitRepository/zeebe/dist/target/camunda-cloud-zeebe-1.4.0-SNAPSHOT.tar.gz
```
### (4). gateway配置文件(application.yaml)如下
```
[root@gateway ~]# cat application.yaml|grep -Ev '^$|#'
zeebe:
  broker:
    gateway:
      enable: true
      network:
        port: 26500
      security:
        enabled: false
    network:
      host: 0.0.0.0
    data:
      directory: data
      logSegmentSize: 128MB
      snapshotPeriod: 15m
    cluster:
      clusterSize: 1
      replicationFactor: 1
      partitionsCount: 1
    threads:
      cpuThreadCount: 2
      ioThreadCount: 2
```
### (5). gateway环境变量配置
```
# 可以为zeebe配置环境变量,也可以配置上面(application.yaml)的配置文件
[root@gateway ~]# cat ~/.bash_profile
export ZEEBE_LOG_LEVEL=debug
export ZEEBE_GATEWAY_NETWORK_PORT=26500
export ZEEBE_GATEWAY_NETWORK_HOST=gateway
export ZEEBE_GATEWAY_CLUSTER_PORT=26502
export ZEEBE_GATEWAY_CLUSTER_CONTACTPOINT=node-1:26502
export ZEEBE_STANDALONE_GATEWAY=true
export ZEEBE_GATEWAY_CLUSTER_HOST=gateway
export ZEEBE_BROKER_GATEWAY_NETWORK_HOST=gateway
export ZEEBE_BROKER_NETWORK_HOST=gateway
```
### (6). node-*配置文件(*代表N个机器)
```
[root@node-1 ~]# cat application.yaml|grep -Ev '^$|#'
zeebe:
  broker:
    gateway:
      enable: true
      network:
        port: 26500
      security:
        enabled: false
    network:
      host: 0.0.0.0
    data:
      directory: data
      logSegmentSize: 128MB
      snapshotPeriod: 15m
    cluster:
      clusterSize: 1
      replicationFactor: 1
      partitionsCount: 1
    threads:
      cpuThreadCount: 2
      ioThreadCount: 2
```
### (7). node-*环境变量配置
```
# 我这里以node-1节点为案例,其余节点配是一样的.

export ZEEBE_LOG_LEVEL=debug

# ***********************************************
# ZEEBE_BROKER_CLUSTER_NODEID每台机器的配置不一样,这是代表节点的唯一id
export ZEEBE_BROKER_CLUSTER_NODEID=0

# ***********************************************
# 配置其它所有的节点
export ZEEBE_BROKER_CLUSTER_INITIALCONTACTPOINTS=node-1:26502,node-2:26502,node-3:26502
export ZEEBE_STANDALONE_GATEWAY=false
export ZEEBE_BROKER_CLUSTER_PARTITIONSCOUNT=2
export ZEEBE_BROKER_CLUSTER_CLUSTERSIZE=3
export ZEEBE_BROKER_CLUSTER_REPLICATIONFACTOR=3
export ZEEBE_BROKER_GATEWAY_NETWORK_HOST=0.0.0.0

# ***********************************************
# 配置本机的hostname
export ZEEBE_BROKER_NETWORK_HOST=node-1
export ZEEBE_BROKER_GATEWAY_CLUSTER_HOST=node-1
```
### (8).  node-*启动
```
2022-09-14 13:42:03.871 [] [main] INFO io.camunda.zeebe.broker.StandaloneBroker - Starting StandaloneBroker v1.4.0-SNAPSHOT using Java 18.0.2.1 on node-1 with PID 10228 (/root/zeebe-node-1.4.0/lib/camunda-cloud-zeebe-1.4.0-SNAPSHOT.jar started by root in /root/zeebe-no
de-1.4.0/bin)
2022-09-14 13:42:03.938 [] [main] DEBUG io.camunda.zeebe.broker.StandaloneBroker - Running with Spring Boot v2.6.3, Spring v5.3.16
2022-09-14 13:42:03.940 [] [main] INFO io.camunda.zeebe.broker.StandaloneBroker - The following profiles are active: broker
2022-09-14 13:42:08.923 [] [main] INFO org.springframework.boot.web.embedded.tomcat.TomcatWebServer - Tomcat initialized with port(s): 9600 (http)
2022-09-14 13:42:08.976 [] [main] INFO org.apache.coyote.http11.Http11NioProtocol - Initializing ProtocolHandler ["http-nio-0.0.0.0-9600"]
2022-09-14 13:42:09.006 [] [main] INFO org.apache.catalina.core.StandardService - Starting service [Tomcat]
2022-09-14 13:42:09.050 [] [main] INFO org.apache.catalina.core.StandardEngine - Starting Servlet engine: [Apache Tomcat/9.0.56]
2022-09-14 13:42:09.736 [] [main] INFO org.apache.catalina.core.ContainerBase.[Tomcat].[localhost].[/] - Initializing Spring embedded WebApplicationContext
2022-09-14 13:42:09.737 [] [main] INFO org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext - Root WebApplicationContext: initialization completed in 5584 ms
2022-09-14 13:42:15.802 [] [main] INFO org.springframework.boot.actuate.endpoint.web.EndpointLinksResolver - Exposing 6 endpoint(s) beneath base path '/actuator'
2022-09-14 13:42:16.045 [] [main] INFO org.apache.coyote.http11.Http11NioProtocol - Starting ProtocolHandler ["http-nio-0.0.0.0-9600"]
2022-09-14 13:42:16.173 [] [main] INFO org.springframework.boot.web.embedded.tomcat.TomcatWebServer - Tomcat started on port(s): 9600 (http) with context path ''
2022-09-14 13:42:16.403 [] [main] INFO io.camunda.zeebe.broker.StandaloneBroker - Started StandaloneBroker in 14.889 seconds (JVM running for 16.867)
2022-09-14 13:42:16.542 [] [main] DEBUG io.camunda.zeebe.broker.system - Initializing system with base path /root/zeebe-node-1.4.0
2022-09-14 13:42:17.281 [] [main] INFO io.camunda.zeebe.broker.system - Version: 1.4.0-SNAPSHOT
2022-09-14 13:42:17.667 [] [main] INFO io.camunda.zeebe.broker.system - Starting broker 0 with configuration {
  "network" : {
    "host" : "node-1",
    "portOffset" : 0,
    "maxMessageSize" : "4MB",
    "advertisedHost" : "node-1",
    "commandApi" : {
      "host" : "node-1",
      "port" : 26501,
      "advertisedHost" : "node-1",
      "advertisedPort" : 26501,
      "address" : "node-1:26501",
      "advertisedAddress" : "node-1:26501"
    },
    "internalApi" : {
      "host" : "node-1",
      "port" : 26502,
      "advertisedHost" : "node-1",
      "advertisedPort" : 26502,
      "address" : "node-1:26502",
      "advertisedAddress" : "node-1:26502"
    },
    "security" : {
      "enabled" : false,
      "certificateChainPath" : null,
      "privateKeyPath" : null
    },
    "maxMessageSizeInBytes" : 4194304
},
  "cluster" : {
    "initialContactPoints" : [ "node-1:26502" ],
    "partitionIds" : [ 1 ],
    "nodeId" : 0,
    "partitionsCount" : 1,
    "replicationFactor" : 1,
    "clusterSize" : 1,
    "clusterName" : "zeebe-cluster",
    "heartbeatInterval" : "PT0.25S",
    "electionTimeout" : "PT2.5S",
    "membership" : {
      "broadcastUpdates" : false,
      "broadcastDisputes" : true,
      "notifySuspect" : false,
      "gossipInterval" : "PT0.25S",
      "gossipFanout" : 2,
      "probeInterval" : "PT1S",
      "probeTimeout" : "PT0.1S",
      "suspectProbes" : 3,
      "failureTimeout" : "PT10S",
      "syncInterval" : "PT10S"
    },
    "raft" : {
      "enablePriorityElection" : true
    },
    "messageCompression" : "NONE"
  },"threads" : {
    "cpuThreadCount" : 2,
    "ioThreadCount" : 2
  },
  "data" : {
    "directory" : "/root/zeebe-node-1.4.0/data",
    "logSegmentSize" : "128MB",
    "snapshotPeriod" : "PT15M",
    "logIndexDensity" : 100,
    "diskUsageMonitoringEnabled" : true,
    "diskUsageReplicationWatermark" : 0.99,
    "diskUsageCommandWatermark" : 0.97,
    "diskUsageMonitoringInterval" : "PT1S",
    "freeDiskSpaceReplicationWatermark" : 527103099,
    "logSegmentSizeInBytes" : 134217728,
    "freeDiskSpaceCommandWatermark" : 1581309297
  },
  "exporters" : { },
"gateway" : {
    "network" : {
      "host" : "0.0.0.0",
      "port" : 26500,
      "minKeepAliveInterval" : "PT30S"
    },
    "cluster" : {
      "contactPoint" : "10.211.55.101:26502",
      "requestTimeout" : "PT15S",
      "clusterName" : "zeebe-cluster",
      "memberId" : "gateway",
      "host" : "node-1",
      "port" : 26502,
      "membership" : {
        "broadcastUpdates" : false,
        "broadcastDisputes" : true,
        "notifySuspect" : false,
        "gossipInterval" : "PT0.25S",
        "gossipFanout" : 2,
        "probeInterval" : "PT1S",
        "probeTimeout" : "PT0.1S",
        "suspectProbes" : 3,
        "failureTimeout" : "PT10S",
        "syncInterval" : "PT10S"
      },
      "security" : {
        "enabled" : false,
        "certificateChainPath" : null,
        "privateKeyPath" : null
      },
	"longPolling" : {
      "enabled" : true
    },
    "interceptors" : [ ],
    "initialized" : true,
    "enable" : true
  },
"backpressure" : {
    "enabled" : true,
    "algorithm" : "VEGAS",
    "aimd" : {
      "requestTimeout" : "PT1S",
      "initialLimit" : 100,
      "minLimit" : 1,
      "maxLimit" : 1000,
      "backoffRatio" : 0.9
    },
    "fixed" : {
      "limit" : 20
    },
    "vegas" : {
      "alpha" : 3,
      "beta" : 6,
      "initialLimit" : 20
    },
    "gradient" : {
      "minLimit" : 10,
      "initialLimit" : 20,
      "rttTolerance" : 2.0
    },
    "gradient2" : {
      "minLimit" : 10,
      "initialLimit" : 20,
      "rttTolerance" : 2.0,
      "longWindow" : 600
    }
  },
"experimental" : {
    "maxAppendsPerFollower" : 2,
    "maxAppendBatchSize" : "32KB",
    "disableExplicitRaftFlush" : false,
    "rocksdb" : {
      "columnFamilyOptions" : { },
      "enableStatistics" : false,
      "memoryLimit" : "512MB",
      "maxOpenFiles" : -1,
      "maxWriteBufferNumber" : 6,
      "minWriteBufferNumberToMerge" : 3,
      "ioRateBytesPerSecond" : 0,
      "disableWal" : false
    },
    "raft" : {
      "requestTimeout" : "PT5S",
      "maxQuorumResponseTimeout" : "PT0S",
      "minStepDownFailureCount" : 3,
      "preferSnapshotReplicationThreshold" : 100
    },
    "partitioning" : {
      "scheme" : "ROUND_ROBIN",
      "fixed" : [ ]
    },
    "queryApi" : {
      "enabled" : false
    },
    "maxAppendBatchSizeInBytes" : 32768
  },
  "executionMetricsExporterEnabled" : false
}
2022-09-14 13:42:17.710 [Broker-0-Startup] [Broker-0-zb-actors-1] DEBUG io.camunda.zeebe.broker.system - Startup was called with context: io.camunda.zeebe.broker.bootstrap.BrokerStartupContextImpl@1ea580e8
2022-09-14 13:42:17.715 [Broker-0-Startup] [Broker-0-zb-actors-1] INFO io.camunda.zeebe.broker.system - Startup Disk Space Usage Monitor
2022-09-14 13:42:17.744 [Broker-0-Startup] [Broker-0-zb-actors-1] INFO io.camunda.zeebe.broker.system - Startup Health Monitor
2022-09-14 13:42:17.747 [Broker-0-Startup] [Broker-0-zb-actors-1] INFO io.camunda.zeebe.broker.system - Startup Broker Admin Interface
2022-09-14 13:42:17.770 [Broker-0-Startup] [Broker-0-zb-actors-1] INFO io.camunda.zeebe.broker.system - Startup Cluster Services (Start)
2022-09-14 13:42:17.850 [Broker-0-Startup] [Broker-0-zb-actors-1] DEBUG io.camunda.zeebe.broker.clustering - Member 0 will contact node: node-1:26502
2022-09-14 13:42:19.743 [] [netty-messaging-event-epoll-server-0] INFO io.atomix.cluster.messaging.impl.NettyMessagingService - TCP server listening for connections on node-1:26502
2022-09-14 13:42:19.774 [] [netty-messaging-event-epoll-server-0] INFO io.atomix.cluster.messaging.impl.NettyMessagingService - Started messaging service bound to [node-1:26502], adve rtising node-1:26502, and using plaintext
2022-09-14 13:42:20.058 [] [netty-unicast-event-nio-client-0] INFO io.atomix.cluster.messaging.impl.NettyUnicastService - UDP server listening for connections on node-1:26502
2022-09-14 13:42:20.060 [] [atomix-cluster-0] INFO io.atomix.cluster.discovery.BootstrapDiscoveryProvider - Local node Node{id=0, address=node-1:26502} joined the bootstrap service
2022-09-14 13:42:20.093 [] [atomix-cluster-0] INFO io.atomix.cluster.protocol.SwimMembershipProtocol - Started
2022-09-14 13:42:20.094 [] [atomix-cluster-0] INFO io.atomix.cluster.impl.DefaultClusterMembershipService - Started cluster membership service for member Member{id=0, address=node-1:26502, properties={}}
2022-09-14 13:42:20.095 [] [atomix-cluster-0] INFO io.atomix.cluster.messaging.impl.DefaultClusterCommunicationService - Started
2022-09-14 13:42:20.098 [] [atomix-cluster-0] INFO io.atomix.cluster.messaging.impl.DefaultClusterEventService - Started
2022-09-14 13:42:20.100 [Broker-0-Startup] [Broker-0-zb-actors-0] INFO io.camunda.zeebe.broker.system - Startup API Messaging Service
2022-09-14 13:42:20.111 [] [netty-messaging-event-epoll-server-0] INFO io.atomix.cluster.messaging.impl.NettyMessagingService - TCP server listening for connections on node-1:26501
2022-09-14 13:42:20.113 [] [netty-messaging-event-epoll-server-0] INFO io.atomix.cluster.messaging.impl.NettyMessagingService - Started messaging service bound to [node-1:26501], advertising node-1:26501, and using plaintext
2022-09-14 13:42:20.121 [Broker-0-Startup] [Broker-0-zb-actors-1] DEBUG io.camunda.zeebe.broker.system - Bound API to [node-1:26501], using advertised address node-1:26501
2022-09-14 13:42:20.122 [Broker-0-Startup] [Broker-0-zb-actors-1] INFO io.camunda.zeebe.broker.system - Startup Command API
2022-09-14 13:42:20.413 [Broker-0-Startup] [Broker-0-zb-actors-1] INFO io.camunda.zeebe.broker.system - Startup Admin API
2022-09-14 13:42:20.419 [Broker-0-Startup] [Broker-0-zb-actors-1] INFO io.camunda.zeebe.broker.system - Startup Subscription API
2022-09-14 13:42:20.487 [Broker-0-Startup] [Broker-0-zb-actors-0] INFO io.camunda.zeebe.broker.system - Startup Leader Management Request Handler
2022-09-14 13:42:20.495 [Broker-0-Startup] [Broker-0-zb-actors-0] INFO io.camunda.zeebe.broker.system - Startup Embedded Gateway
2022-09-14 13:42:21.240 [Broker-0-Startup] [Broker-0-zb-actors-0] INFO io.camunda.zeebe.broker.system - Startup Partition Manager
2022-09-14 13:42:21.439 [] [Thread-12] INFO io.atomix.raft.partition.impl.RaftPartitionServer - RaftPartitionServer{raft-partition-partition-1} - Starting server for partition PartitionId{id=1, group=raft-partition}
2022-09-14 13:42:21.515 [GatewayTopologyManager] [Broker-0-zb-actors-0] DEBUG io.camunda.zeebe.gateway - Received metadata change from Broker 0, partitions {}, terms {} and health {}.
2022-09-14 13:42:22.176 [] [raft-server-0-raft-partition-partition-1] INFO io.atomix.raft.impl.RaftContext - RaftServer{raft-partition-partition-1} - Transitioning to FOLLOWER
2022-09-14 13:42:22.229 [] [raft-server-0-raft-partition-partition-1] INFO io.atomix.raft.roles.FollowerRole - RaftServer{raft-partition-partition-1}{role=FOLLOWER} - Single member cluster. Transitioning directly to candidate.
2022-09-14 13:42:22.231 [] [raft-server-0-raft-partition-partition-1] INFO io.atomix.raft.impl.RaftContext - RaftServer{raft-partition-partition-1} - Transitioning to CANDIDATE
2022-09-14 13:42:22.232 [] [raft-server-0-raft-partition-partition-1] WARN io.atomix.utils.event.ListenerRegistry - Listener io.atomix.raft.roles.FollowerRole$$Lambda$1167/0x00000008012c5188@5c98ce41 not registered
2022-09-14 13:42:22.268 [] [raft-server-0-raft-partition-partition-1] INFO io.atomix.raft.roles.CandidateRole - RaftServer{raft-partition-partition-1}{role=CANDIDATE} - Single member cluster. Transitioning directly to leader.
2022-09-14 13:42:22.319 [] [raft-server-0-raft-partition-partition-1] INFO io.atomix.raft.impl.RaftContext - RaftServer{raft-partition-partition-1} - Transitioning to LEADER
2022-09-14 13:42:22.420 [] [raft-server-0-raft-partition-partition-1] INFO io.atomix.raft.impl.RaftContext - RaftServer{raft-partition-partition-1} - Found leader 0
2022-09-14 13:42:22.461 [] [raft-server-0-raft-partition-partition-1] INFO io.atomix.raft.impl.RaftContext - RaftServer{raft-partition-partition-1} - Setting firstCommitIndex to 1. RaftServer is ready only after it has committed events upto this index
2022-09-14 13:42:22.492 [] [raft-server-0-raft-partition-partition-1] INFO io.atomix.raft.impl.DefaultRaftServer - RaftServer{raft-partition-partition-1} - Server join completed. Waitingfor the server to be READY
2022-09-14 13:42:22.494 [] [raft-server-0-raft-partition-partition-1] INFO io.atomix.raft.partition.impl.RaftPartitionServer - RaftPartitionServer{raft-partition-partition-1} - Successfully started server for partition PartitionId{id=1, group=raft-partition} in 1054ms
2022-09-14 13:42:22.495 [] [raft-server-0-raft-partition-partition-1] INFO io.atomix.raft.partition.RaftPartitionGroup - Started
2022-09-14 13:42:22.496 [] [raft-server-0-raft-partition-partition-1] INFO io.camunda.zeebe.broker.partitioning.PartitionManagerImpl - Registering Partition Manager
2022-09-14 13:42:22.507 [] [raft-server-0-raft-partition-partition-1] INFO io.camunda.zeebe.broker.partitioning.PartitionManagerImpl - Starting partitions
2022-09-14 13:42:23.364 [Broker-0-HealthCheckService] [Broker-0-zb-actors-0] DEBUG io.camunda.zeebe.broker.system - Detected 'UNHEALTHY' components. The current health status of components: [Partition-1{status=UNHEALTHY, issue='Components are not yet initialized'}]
2022-09-14 13:42:23.370 [Broker-0-ZeebePartition-1] [Broker-0-zb-actors-1] DEBUG io.camunda.zeebe.broker.system - Startup was called with context: io.camunda.zeebe.broker.system.partitions.PartitionStartupAndTransitionContextImpl@7fb2d7eb
2022-09-14 13:42:23.371 [Broker-0-ZeebePartition-1] [Broker-0-zb-actors-1] INFO io.camunda.zeebe.broker.system - Startup LogDeletionService
2022-09-14 13:42:23.375 [Broker-0-ZeebePartition-1] [Broker-0-zb-actors-1] INFO io.camunda.zeebe.broker.system - Startup RocksDB metric timer
2022-09-14 13:42:23.445 [Broker-0-ZeebePartition-1] [Broker-0-zb-actors-1] DEBUG io.camunda.zeebe.broker.system - Finished startup process
```
### (9). gateway启动
```
2022-09-14 13:42:21.366 [] [main] INFO io.camunda.zeebe.gateway.StandaloneGateway - Starting StandaloneGateway v1.4.0-SNAPSHOT using Java 18.0.2.1 on gateway with PID 10174 (/root/zeebe-gateway-1.4.0/lib/camunda-cloud-zeebe-1.4.0-SNAPSHOT.jar started by root in /root/zeebe-gateway-1.4.0/bin)
2022-09-14 13:42:21.483 [] [main] DEBUG io.camunda.zeebe.gateway.StandaloneGateway - Running with Spring Boot v2.6.3, Spring v5.3.16
2022-09-14 13:42:21.490 [] [main] INFO io.camunda.zeebe.gateway.StandaloneGateway - The following profiles are active: gateway
2022-09-14 13:42:32.430 [] [main] INFO org.springframework.boot.web.embedded.tomcat.TomcatWebServer - Tomcat initialized with port(s): 9600 (http)
2022-09-14 13:42:32.524 [] [main] INFO org.apache.coyote.http11.Http11NioProtocol - Initializing ProtocolHandler ["http-nio-0.0.0.0-9600"]
2022-09-14 13:42:32.526 [] [main] INFO org.apache.catalina.core.StandardService - Starting service [Tomcat]
2022-09-14 13:42:32.527 [] [main] INFO org.apache.catalina.core.StandardEngine - Starting Servlet engine: [Apache Tomcat/9.0.56]
2022-09-14 13:42:33.222 [] [main] INFO org.apache.catalina.core.ContainerBase.[Tomcat].[localhost].[/] - Initializing Spring embedded WebApplicationContext
2022-09-14 13:42:33.223 [] [main] INFO org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext - Root WebApplicationContext: initialization completed in 11244 ms
2022-09-14 13:42:37.601 [] [main] INFO org.springframework.boot.actuate.endpoint.web.EndpointLinksResolver - Exposing 5 endpoint(s) beneath base path '/actuator'
2022-09-14 13:42:37.724 [] [main] INFO org.apache.coyote.http11.Http11NioProtocol - Starting ProtocolHandler ["http-nio-0.0.0.0-9600"]
2022-09-14 13:42:37.839 [] [main] INFO org.springframework.boot.web.embedded.tomcat.TomcatWebServer - Tomcat started on port(s): 9600 (http) with context path ''
2022-09-14 13:42:37.921 [] [main] INFO io.camunda.zeebe.gateway.StandaloneGateway - Started StandaloneGateway in 20.225 seconds (JVM running for 26.755)
2022-09-14 13:42:37.948 [] [main] INFO io.camunda.zeebe.gateway - Version: 1.4.0-SNAPSHOT
2022-09-14 13:42:38.001 [] [main] INFO io.camunda.zeebe.gateway - Starting standalone gateway with configuration {
  "network" : {
    "host" : "gateway",
    "port" : 26500,
    "minKeepAliveInterval" : "PT30S"
  },
  "cluster" : {
    "contactPoint" : "node-1:26502",
    "requestTimeout" : "PT15S",
    "clusterName" : "zeebe-cluster",
    "memberId" : "gateway",
    "host" : "gateway",
    "port" : 26502,
    "membership" : {
      "broadcastUpdates" : false,
      "broadcastDisputes" : true,
      "notifySuspect" : false,
      "gossipInterval" : "PT0.25S",
      "gossipFanout" : 2,
      "probeInterval" : "PT1S",
      "probeTimeout" : "PT0.1S",
      "suspectProbes" : 3,
      "failureTimeout" : "PT10S",
      "syncInterval" : "PT10S"
    },
    "security" : {
      "enabled" : false,
      "certificateChainPath" : null,
      "privateKeyPath" : null
    },
    "messageCompression" : "NONE"
  },
  "threads" : {
    "managementThreads" : 1
  },
  "security" : {
    "enabled" : false,
    "certificateChainPath" : null,
    "privateKeyPath" : null
  },
  "longPolling" : {
    "enabled" : true
  },
  "interceptors" : [ ],
  "initialized" : true
}
2022-09-14 13:42:38.634 [] [netty-messaging-event-epoll-server-0] INFO io.atomix.cluster.messaging.impl.NettyMessagingService - TCP server listening for connections on gateway:26502
2022-09-14 13:42:38.643 [] [netty-messaging-event-epoll-server-0] INFO io.atomix.cluster.messaging.impl.NettyMessagingService - Started messaging service bound to [gateway:26502], advertising gateway:26502, and using plaintext
2022-09-14 13:42:38.746 [] [netty-unicast-event-nio-client-0] INFO io.atomix.cluster.messaging.impl.NettyUnicastService - UDP server listening for connections on 0.0.0.0:26502
2022-09-14 13:42:38.772 [] [atomix-cluster-0] INFO io.atomix.cluster.discovery.BootstrapDiscoveryProvider - Local node Node{id=gateway, address=gateway:26502} joined the bootstrap service
2022-09-14 13:42:38.848 [] [atomix-cluster-0] INFO io.atomix.cluster.protocol.SwimMembershipProtocol - Started
2022-09-14 13:42:38.848 [] [atomix-cluster-0] INFO io.atomix.cluster.impl.DefaultClusterMembershipService - Started cluster membership service for member Member{id=gateway, address=gateway:26502, properties={event-service-topics-subscribed=KIIDAGpvYnNBdmFpbGFibOU=}}
2022-09-14 13:42:38.853 [] [atomix-cluster-0] INFO io.atomix.cluster.messaging.impl.DefaultClusterCommunicationService - Started
2022-09-14 13:42:38.875 [] [atomix-cluster-0] INFO io.atomix.cluster.messaging.impl.DefaultClusterEventService - Started
2022-09-14 13:42:41.392 [GatewayTopologyManager] [gateway-scheduler-zb-actors-0] DEBUG io.camunda.zeebe.gateway - Received new broker BrokerInfo{nodeId=0, partitionsCount=1, clusterSize=1, replicationFactor=1, partitionRoles={1=LEADER}, partitionLeaderTerms={1=2}, partitionHealthStatuses={1=HEALTHY}, version=1.4.0-SNAPSHOT}.
2022-09-14 13:42:40.012 [GatewayTopologyManager] [gateway-scheduler-zb-actors-0] DEBUG io.camunda.zeebe.gateway - Received REACHABILITY_CHANGED for broker 0, do nothing.
2022-09-14 13:42:42.076 [GatewayTopologyManager] [gateway-scheduler-zb-actors-0] DEBUG io.camunda.zeebe.gateway - Received REACHABILITY_CHANGED for broker 0, do nothing.
2022-09-14 13:42:44.195 [GatewayTopologyManager] [gateway-scheduler-zb-actors-0] DEBUG io.camunda.zeebe.gateway - Received metadata change from Broker 0, partitions {1=LEADER}, terms {1=4} and health {1=HEALTHY}.
2022-09-14 13:42:46.696 [GatewayTopologyManager] [gateway-scheduler-zb-actors-0] DEBUG io.camunda.zeebe.gateway - Received new broker BrokerInfo{nodeId=1, partitionsCount=2, clusterSize=3, replicationFactor=3, partitionRoles={1=FOLLOWER, 2=LEADER}, partitionLeaderTerms={2=1}, partitionHealthStatuses={1=HEALTHY, 2=HEALTHY}, version=1.4.0-SNAPSHOT}.
2022-09-14 13:42:46.699 [GatewayTopologyManager] [gateway-scheduler-zb-actors-0] DEBUG io.camunda.zeebe.gateway - Received new broker BrokerInfo{nodeId=2, partitionsCount=2, clusterSize=3, replicationFactor=3, partitionRoles={}, partitionLeaderTerms={}, partitionHealthStatuses={1=UNHEALTHY, 2=UNHEALTHY}, version=1.4.0-SNAPSHOT}.
2022-09-14 13:42:49.261 [GatewayTopologyManager] [gateway-scheduler-zb-actors-0] DEBUG io.camunda.zeebe.gateway - Received metadata change from Broker 0, partitions {1=LEADER}, terms {1=6} and health {1=HEALTHY}.
2022-09-14 13:42:50.273 [GatewayTopologyManager] [gateway-scheduler-zb-actors-0] DEBUG io.camunda.zeebe.gateway - Received metadata change from Broker 2, partitions {1=FOLLOWER, 2=FOLLOWER}, terms {} and health {1=HEALTHY, 2=HEALTHY}.
2022-09-14 13:42:55.392 [GatewayTopologyManager] [gateway-scheduler-zb-actors-0] DEBUG io.camunda.zeebe.gateway - Received metadata change from Broker 0, partitions {1=LEADER}, terms {1=8} and health {1=HEALTHY}.
2022-09-14 13:43:00.522 [GatewayTopologyManager] [gateway-scheduler-zb-actors-0] DEBUG io.camunda.zeebe.gateway - Received metadata change from Broker 1, partitions {1=LEADER, 2=LEADER}, terms {1=9, 2=1} and health {1=HEALTHY, 2=HEALTHY}.
2022-09-14 13:43:01.535 [GatewayTopologyManager] [gateway-scheduler-zb-actors-0] DEBUG io.camunda.zeebe.gateway - Received metadata change from Broker 0, partitions {1=LEADER}, terms {1=10} and health {1=HEALTHY}.
2022-09-14 13:43:04.574 [GatewayTopologyManager] [gateway-scheduler-zb-actors-0] DEBUG io.camunda.zeebe.gateway - Received metadata change from Broker 1, partitions {1=FOLLOWER, 2=LEADER}, terms {2=1} and health {1=HEALTHY, 2=HEALTHY}.
2022-09-14 13:43:06.797 [GatewayTopologyManager] [gateway-scheduler-zb-actors-0] DEBUG io.camunda.zeebe.gateway - Received metadata change from Broker 0, partitions {1=LEADER}, terms {1=12} and health {1=HEALTHY}.
```
### (10). java代码验证集群状态
```
package io.camunda.zeebe.example.cluster;

import io.camunda.zeebe.client.ZeebeClient;
import io.camunda.zeebe.client.ZeebeClientBuilder;
import io.camunda.zeebe.client.api.response.Topology;

public final class TopologyViewer {

  public static void main(final String[] args) {
    final String defaultAddress = "10.211.55.100:26500";
    //    final String defaultAddress = "localhost:26500";
    final String envVarAddress = System.getenv("ZEEBE_ADDRESS");

    final ZeebeClientBuilder clientBuilder;
    final String contactPoint;
    if (envVarAddress != null) {
      /* Connect to Camunda Cloud Cluster, assumes that credentials are set in environment variables.
       * See JavaDoc on class level for details
       */
      contactPoint = envVarAddress;
      clientBuilder = ZeebeClient.newClientBuilder().gatewayAddress(envVarAddress);
    } else {
      // connect to local deployment; assumes that authentication is disabled
      contactPoint = defaultAddress;
      clientBuilder = ZeebeClient.newClientBuilder().gatewayAddress(defaultAddress).usePlaintext();
    }

    try (final ZeebeClient client = clientBuilder.build()) {
      System.out.println("Requesting topology with initial contact point " + contactPoint);

      final Topology topology = client.newTopologyRequest().send().join();

      System.out.println("Topology:");
      topology
          .getBrokers()
          .forEach(
              b -> {
                System.out.println("    " + b.getAddress());
                b.getPartitions()
                    .forEach(
                        p ->
                            System.out.println(
                                "      " + p.getPartitionId() + " - " + p.getRole()));
              });
      System.out.println("Done.");
    }
  }
}
```
### (11). 查看结果
> 运行后结果如下:

```
Connected to the target VM, address: '127.0.0.1:53274', transport: 'socket'
Requesting topology with initial contact point 10.211.55.100:26500
Topology:
    node-1:26501
      1 - LEADER
    node-2:26501
      1 - FOLLOWER
      2 - LEADER
    node-3:26501
      1 - FOLLOWER
      2 - FOLLOWER
Done.
```
### (12). 总结
总的来说,Zeebe的集群还是比较简单的,但是,我第一次搭建却花费了不少时间,原因在于:有几个环境变量是隐式的.  