---
layout: post
title: 'Zeebe Broker安装(二)' 
date: 2022-02-02
author: 李新
tags:  Zeebe
---

### (1). 概述
在这一篇主要对Zeebe Broker安装.

### (2). 安装前提(略)
+ ElasticSearch,因为Zeebe默认的Export为ElasticSearch.  
+ Kibana(可选)

### (3). Zeebe安装
```
# 1. 进入dist二进制程序目录
lixin-macbook:zeebe lixin$ cd dist/target/
# 2. 解压
lixin-macbook:target lixin$ tar -zxvf camunda-cloud-zeebe-1.3.0-SNAPSHOT.tar.gz 
# 3. 移动目录
lixin-macbook:target lixin$ mv camunda-cloud-zeebe-1.3.0-SNAPSHOT ~/Developer/camunda-cloud-zeebe-1.3.0
```
### (4). Zeebe配置
```
lixin-macbook:target lixin$ cd ~/Developer/camunda-cloud-zeebe-1.3.0/

# 1. 配置
lixin-macbook:camunda-cloud-zeebe-1.3.0 lixin$ vi config/application.yaml 
```
### (5). Zeebe配置(application.yaml )
```
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
      ioThreadCount: 2
    exporters:      
      elasticsearch: 
        className: io.camunda.zeebe.exporter.ElasticsearchExporter
        args:
          url: http://localhost:9200
          requestTimeoutMs: 1000
```
### (6). Zeebe启动
````
lixin-macbook:camunda-cloud-zeebe-1.3.0 lixin$ ./bin/broker 
  ______  ______   ______   ____    ______     ____    _____     ____    _  __  ______   _____  
 |___  / |  ____| |  ____| |  _ \  |  ____|   |  _ \  |  __ \   / __ \  | |/ / |  ____| |  __ \ 
    / /  | |__    | |__    | |_) | | |__      | |_) | | |__) | | |  | | | ' /  | |__    | |__) |
   / /   |  __|   |  __|   |  _ <  |  __|     |  _ <  |  _  /  | |  | | |  <   |  __|   |  _  / 
  / /__  | |____  | |____  | |_) | | |____    | |_) | | | \ \  | |__| | | . \  | |____  | | \ \ 
 /_____| |______| |______| |____/  |______|   |____/  |_|  \_\  \____/  |_|\_\ |______| |_|  \_\
                                                                                                
2022-02-02 22:27:42.009 [] [main] INFO io.camunda.zeebe.broker.StandaloneBroker - Starting StandaloneBroker v1.3.0-SNAPSHOT using Java 11.0.9 on lixin-macbook.local with PID 3664 (/Users/lixin/Developer/camunda-cloud-zeebe-1.3.0/lib/camunda-cloud-zeebe-1.3.0-SNAPSHOT.jar started by lixin in /Users/lixin/Developer/camunda-cloud-zeebe-1.3.0)
2022-02-02 22:27:42.025 [] [main] INFO io.camunda.zeebe.broker.StandaloneBroker - No active profile set, falling back to default profiles: default
2022-02-02 22:27:44.561 [] [main] INFO org.springframework.boot.web.embedded.tomcat.TomcatWebServer - Tomcat initialized with port(s): 9600 (http)
2022-02-02 22:27:44.587 [] [main] INFO org.apache.coyote.http11.Http11NioProtocol - Initializing ProtocolHandler ["http-nio-0.0.0.0-9600"]
2022-02-02 22:27:44.588 [] [main] INFO org.apache.catalina.core.StandardService - Starting service [Tomcat]
2022-02-02 22:27:44.588 [] [main] INFO org.apache.catalina.core.StandardEngine - Starting Servlet engine: [Apache Tomcat/9.0.52]
2022-02-02 22:27:44.916 [] [main] INFO org.apache.catalina.core.ContainerBase.[Tomcat].[localhost].[/] - Initializing Spring embedded WebApplicationContext
2022-02-02 22:27:44.916 [] [main] INFO org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext - Root WebApplicationContext: initialization completed in 2777 ms
2022-02-02 22:27:45.947 [] [main] INFO org.springframework.boot.actuate.endpoint.web.EndpointLinksResolver - Exposing 4 endpoint(s) beneath base path '/actuator'
2022-02-02 22:27:45.994 [] [main] INFO org.apache.coyote.http11.Http11NioProtocol - Starting ProtocolHandler ["http-nio-0.0.0.0-9600"]
2022-02-02 22:27:46.049 [] [main] INFO org.springframework.boot.web.embedded.tomcat.TomcatWebServer - Tomcat started on port(s): 9600 (http) with context path ''
2022-02-02 22:27:46.079 [] [main] INFO io.camunda.zeebe.broker.StandaloneBroker - Started StandaloneBroker in 5.061 seconds (JVM running for 6.986)
2022-02-02 22:27:46.492 [] [main] INFO io.camunda.zeebe.broker.system - Version: 1.3.0-SNAPSHOT
2022-02-02 22:27:46.598 [] [main] INFO io.camunda.zeebe.broker.system - Starting broker 0 with configuration {
  "network" : {
    "host" : "0.0.0.0",
    "portOffset" : 0,
    "maxMessageSize" : "4MB",
    "advertisedHost" : "0.0.0.0",
    "commandApi" : {
      "host" : "0.0.0.0",
      "port" : 26501,
      "advertisedHost" : "0.0.0.0",
      "advertisedPort" : 26501,
      "address" : "0.0.0.0:26501",
      "advertisedAddress" : "0.0.0.0:26501"
    },
    "internalApi" : {
      "host" : "0.0.0.0",
      "port" : 26502,
      "advertisedHost" : "0.0.0.0",
      "advertisedPort" : 26502,
      "address" : "0.0.0.0:26502",
      "advertisedAddress" : "0.0.0.0:26502"
    },
    "monitoringApi" : {
      "host" : "0.0.0.0",
      "port" : 9600,
      "advertisedHost" : "0.0.0.0",
      "advertisedPort" : 9600,
      "address" : "0.0.0.0:9600",
      "advertisedAddress" : "0.0.0.0:9600"
    },
    "maxMessageSizeInBytes" : 4194304
  },
  "cluster" : {
    "initialContactPoints" : [ ],
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
    }
  },
  "threads" : {
    "cpuThreadCount" : 2,
    "ioThreadCount" : 2
  },
  "data" : {
    "directory" : "/Users/lixin/Developer/camunda-cloud-zeebe-1.3.0/data",
    "logSegmentSize" : "128MB",
    "snapshotPeriod" : "PT15M",
    "logIndexDensity" : 100,
    "diskUsageMonitoringEnabled" : true,
    "diskUsageReplicationWatermark" : 0.99,
    "diskUsageCommandWatermark" : 0.97,
    "diskUsageMonitoringInterval" : "PT1S",
    "logSegmentSizeInBytes" : 134217728,
    "freeDiskSpaceCommandWatermark" : 15002041098,
    "freeDiskSpaceReplicationWatermark" : 5000680366
  },
  "exporters" : {
    "elasticsearch" : {
      "jarPath" : null,
      "className" : "io.camunda.zeebe.exporter.ElasticsearchExporter",
      "args" : {
        "url" : "http://localhost:9200",
        "requestTimeoutMs" : 1000
      },
      "external" : false
    }
  },
  "gateway" : {
    "network" : {
      "host" : "0.0.0.0",
      "port" : 26500,
      "minKeepAliveInterval" : "PT30S"
    },
    "cluster" : {
      "contactPoint" : "0.0.0.0:26502",
      "requestTimeout" : "PT15S",
      "clusterName" : "zeebe-cluster",
      "memberId" : "gateway",
      "host" : "0.0.0.0",
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
      }
    },
    "threads" : {
      "managementThreads" : 1
    },
    "monitoring" : {
      "enabled" : false,
      "host" : "0.0.0.0",
      "port" : 9600
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
    "enablePriorityElection" : false,
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
      "minStepDownFailureCount" : 3
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
2022-02-02 22:27:46.617 [] [main] INFO io.camunda.zeebe.broker.system - Bootstrap Broker-0 [1/10]: Migrated Startup Steps
2022-02-02 22:27:46.621 [Broker-0-Startup] [Broker-0-zb-actors-1] INFO io.camunda.zeebe.broker.system - Startup monitoring services
2022-02-02 22:27:46.640 [Broker-0-Startup] [Broker-0-zb-actors-1] INFO io.camunda.zeebe.broker.system - Startup Cluster Services (Creation)
2022-02-02 22:27:46.993 [] [main] INFO io.camunda.zeebe.broker.system - Bootstrap Broker-0 [2/10]: command api transport and handler
2022-02-02 22:27:47.180 [] [netty-messaging-event-nio-server-0] INFO io.atomix.cluster.messaging.impl.NettyMessagingService - TCP server listening for connections on 0.0.0.0:26501
2022-02-02 22:27:47.184 [] [netty-messaging-event-nio-server-0] INFO io.atomix.cluster.messaging.impl.NettyMessagingService - Started
2022-02-02 22:27:47.251 [] [main] INFO io.camunda.zeebe.broker.system - Bootstrap Broker-0 [3/10]: subscription api
2022-02-02 22:27:47.293 [] [main] INFO io.camunda.zeebe.broker.system - Bootstrap Broker-0 [4/10]: cluster services
2022-02-02 22:27:47.296 [] [netty-messaging-event-nio-server-0] INFO io.atomix.cluster.messaging.impl.NettyMessagingService - TCP server listening for connections on 0.0.0.0:26502
2022-02-02 22:27:47.297 [] [netty-messaging-event-nio-server-0] INFO io.atomix.cluster.messaging.impl.NettyMessagingService - Started
2022-02-02 22:27:47.320 [] [netty-unicast-event-nio-client-0] INFO io.atomix.cluster.messaging.impl.NettyUnicastService - UDP server listening for connections on 0.0.0.0:26502
2022-02-02 22:27:47.321 [] [atomix-cluster-0] INFO io.atomix.cluster.discovery.BootstrapDiscoveryProvider - Local node Node{id=0, address=0.0.0.0:26502} joined the bootstrap service
2022-02-02 22:27:47.338 [] [atomix-cluster-0] INFO io.atomix.cluster.protocol.SwimMembershipProtocol - Started
2022-02-02 22:27:47.342 [] [atomix-cluster-0] INFO io.atomix.cluster.impl.DefaultClusterMembershipService - Started cluster membership service for member Member{id=0, address=0.0.0.0:26502, properties={}}
2022-02-02 22:27:47.345 [] [atomix-cluster-0] INFO  io.atomix.cluster.messaging.impl.DefaultClusterCommunicationService - Started
2022-02-02 22:27:47.348 [] [atomix-cluster-0] INFO io.atomix.cluster.messaging.impl.DefaultClusterEventService - Started
2022-02-02 22:27:47.348 [] [main] INFO io.camunda.zeebe.broker.system - Bootstrap Broker-0 [5/10]: embedded gateway
2022-02-02 22:27:47.652 [] [main] INFO io.camunda.zeebe.broker.system - Bootstrap Broker-0 [6/10]: disk space monitor
2022-02-02 22:27:47.656 [] [main] INFO io.camunda.zeebe.broker.system - Bootstrap Broker-0 [7/10]: leader management request handler
2022-02-02 22:27:47.661 [] [main] INFO  io.camunda.zeebe.broker.system - Bootstrap Broker-0 [8/10]: zeebe partitions
2022-02-02 22:27:47.730 [] [main] INFO  io.atomix.raft.partition.impl.RaftPartitionServer - RaftPartitionServer{raft-partition-partition-1} - Starting server for partition PartitionId{id=1, group=raft-partition}
2022-02-02 22:27:48.008 [] [raft-server-0-raft-partition-partition-1] INFO  io.atomix.raft.impl.RaftContext - RaftServer{raft-partition-partition-1} - Transitioning to FOLLOWER
2022-02-02 22:27:48.016 [] [raft-server-0-raft-partition-partition-1] INFO  io.atomix.raft.roles.FollowerRole - RaftServer{raft-partition-partition-1}{role=FOLLOWER} - Single member cluster. Transitioning directly to candidate.
2022-02-02 22:27:48.018 [] [raft-server-0-raft-partition-partition-1] INFO  io.atomix.raft.impl.RaftContext - RaftServer{raft-partition-partition-1} - Transitioning to CANDIDATE
2022-02-02 22:27:48.020 [] [raft-server-0-raft-partition-partition-1] WARN  io.atomix.utils.event.ListenerRegistry - Listener io.atomix.raft.roles.FollowerRole$$Lambda$1100/0x000000080080dc40@3f68b195 not registered
2022-02-02 22:27:48.026 [] [raft-server-0-raft-partition-partition-1] INFO io.atomix.raft.roles.CandidateRole - RaftServer{raft-partition-partition-1}{role=CANDIDATE} - Single member cluster. Transitioning directly to leader.
2022-02-02 22:27:48.051 [] [raft-server-0-raft-partition-partition-1] INFO io.atomix.raft.impl.RaftContext - RaftServer{raft-partition-partition-1} - Transitioning to LEADER
2022-02-02 22:27:48.076 [] [raft-server-0-raft-partition-partition-1] INFO io.atomix.raft.impl.RaftContext - RaftServer{raft-partition-partition-1} - Found leader 0
2022-02-02 22:27:48.108 [] [raft-server-0-raft-partition-partition-1] INFO  io.atomix.raft.impl.RaftContext - RaftServer{raft-partition-partition-1} - Setting firstCommitIndex to 2. RaftServer is ready only after it has committed events upto this index
2022-02-02 22:27:48.118 [] [raft-server-0-raft-partition-partition-1] INFO io.atomix.raft.impl.DefaultRaftServer - RaftServer{raft-partition-partition-1} - Server join completed. Waiting for the server to be READY
2022-02-02 22:27:48.120 [] [raft-server-0-raft-partition-partition-1] INFO io.atomix.raft.partition.impl.RaftPartitionServer - RaftPartitionServer{raft-partition-partition-1} - Successfully started server for partition PartitionId{id=1, group=raft-partition} in 389ms
2022-02-02 22:27:48.121 [] [raft-server-0-raft-partition-partition-1] INFO  io.atomix.raft.partition.RaftPartitionGroup - Started
2022-02-02 22:27:48.122 [] [raft-server-0-raft-partition-partition-1] INFO  io.camunda.zeebe.broker.partitioning.PartitionManagerImpl - Registering Partition Manager
2022-02-02 22:27:48.130 [] [raft-server-0-raft-partition-partition-1] INFO  io.camunda.zeebe.broker.partitioning.PartitionManagerImpl - Starting partitions
2022-02-02 22:27:48.802 [Broker-0-ZeebePartition-1] [Broker-0-zb-actors-0] INFO  io.camunda.zeebe.broker.system - Startup LogDeletionService
2022-02-02 22:27:48.804 [Broker-0-ZeebePartition-1] [Broker-0-zb-actors-0] INFO  io.camunda.zeebe.broker.system - Startup RocksDB metric timer
2022-02-02 22:27:48.809 [Broker-0-ZeebePartition-1] [Broker-0-zb-actors-0] INFO  io.camunda.zeebe.broker.system - Transition to LEADER on term 2 requested.
2022-02-02 22:27:48.815 [Broker-0-ZeebePartition-1] [Broker-0-zb-actors-0] INFO  io.camunda.zeebe.broker.system - Transition to LEADER on term 2 starting
2022-02-02 22:27:48.816 [Broker-0-ZeebePartition-1] [Broker-0-zb-actors-0] INFO  io.camunda.zeebe.broker.system - Transition to LEADER on term 2 - executing LogStorage
2022-02-02 22:27:48.821 [Broker-0-ZeebePartition-1] [Broker-0-zb-actors-0] INFO  io.camunda.zeebe.broker.system - Transition to LEADER on term 2 - executing LogStream
2022-02-02 22:27:48.829 [Broker-0-ZeebePartition-1] [Broker-0-zb-actors-0] INFO  io.camunda.zeebe.broker.system - Transition to LEADER on term 2 - executing ZeebeDb
2022-02-02 22:27:49.033 [Broker-0-ZeebePartition-1] [Broker-0-zb-actors-1] INFO io.camunda.zeebe.broker.system - Transition to LEADER on term 2 - executing QueryService
2022-02-02 22:27:49.199 [Broker-0-ZeebePartition-1] [Broker-0-zb-actors-1] INFO  org.camunda.feel.FeelEngine - Engine created. [value-mapper: CompositeValueMapper(List(io.camunda.zeebe.el.impl.feel.MessagePackValueMapper@2f7726ac)), function-provider: io.camunda.zeebe.el.impl.feel.FeelFunctionProvider@715a3f87, clock: io.camunda.zeebe.el.impl.ZeebeFeelEngineClock@4c4ff204, configuration: Configuration(false)]
2022-02-02 22:27:49.395 [Broker-0-ZeebePartition-1] [Broker-0-zb-actors-1] INFO  io.camunda.zeebe.broker.system - Transition to LEADER on term 2 - executing StreamProcessor
2022-02-02 22:27:49.588 [Broker-0-StreamProcessor-1] [Broker-0-zb-actors-0] INFO org.camunda.feel.FeelEngine - Engine created. [value-mapper: CompositeValueMapper(List(io.camunda.zeebe.el.impl.feel.MessagePackValueMapper@64b91609)), function-provider: io.camunda.zeebe.el.impl.feel.FeelFunctionProvider@336a6d26, clock: io.camunda.zeebe.el.impl.ZeebeFeelEngineClock@78b87586, configuration: Configuration(false)]
2022-02-02 22:27:49.633 [Broker-0-StreamProcessor-1] [Broker-0-zb-actors-0] INFO io.camunda.zeebe.logstreams - Recovered state of partition 1 from snapshot at position -1
2022-02-02 22:27:49.656 [Broker-0-StreamProcessor-1] [Broker-0-zb-actors-0] INFO  org.camunda.feel.FeelEngine - Engine created. [value-mapper: CompositeValueMapper(List(io.camunda.zeebe.el.impl.feel.MessagePackValueMapper@3a5c6ee9)), function-provider: io.camunda.zeebe.el.impl.feel.FeelFunctionProvider@65026e2, clock: io.camunda.zeebe.el.impl.ZeebeFeelEngineClock@2e281773, configuration: Configuration(false)]
2022-02-02 22:27:49.687 [Broker-0-StreamProcessor-1] [Broker-0-zb-actors-0] INFO org.camunda.feel.FeelEngine - Engine created. [value-mapper: CompositeValueMapper(List(io.camunda.zeebe.el.impl.feel.MessagePackValueMapper@77189abd)), function-provider: io.camunda.zeebe.el.impl.feel.FeelFunctionProvider@34019e2b, clock: io.camunda.zeebe.el.impl.ZeebeFeelEngineClock@76d46aa, configuration: Configuration(false)]
2022-02-02 22:27:49.693 [Broker-0-StreamProcessor-1] [Broker-0-zb-actors-0] INFO org.camunda.feel.FeelEngine - Engine created. [value-mapper: CompositeValueMapper(List(io.camunda.zeebe.el.impl.feel.MessagePackValueMapper@74c7afb5)), function-provider: io.camunda.zeebe.el.impl.feel.FeelFunctionProvider@2e05cec1, clock: io.camunda.zeebe.el.impl.ZeebeFeelEngineClock@331ffe78, configuration: Configuration(false)]
2022-02-02 22:27:49.845 [Broker-0-StreamProcessor-1] [Broker-0-zb-actors-0] INFO io.camunda.zeebe.processor - Processor starts replay of events. [snapshot-position: -1, replay-mode: PROCESSING]
2022-02-02 22:27:49.845 [Broker-0-StreamProcessor-1] [Broker-0-zb-actors-0] INFO  io.camunda.zeebe.processor - Processor finished replay at event position -1
2022-02-02 22:27:49.847 [Broker-0-ZeebePartition-1] [Broker-0-zb-actors-1] INFO  io.camunda.zeebe.broker.system - Transition to LEADER on term 2 - executing SnapshotDirector
2022-02-02 22:27:49.849 [Broker-0-StreamProcessor-1] [Broker-0-zb-actors-0] INFO io.camunda.zeebe.engine.state.migration - Starting processing of migration tasks (use LogLevel.DEBUG for more details) ... 
2022-02-02 22:27:49.857 [Broker-0-ZeebePartition-1] [Broker-0-zb-actors-1] INFO  io.camunda.zeebe.broker.system - Transition to LEADER on term 2 - executing ExporterDirector
2022-02-02 22:27:49.884 [Broker-0-StreamProcessor-1] [Broker-0-zb-actors-0] INFO  io.camunda.zeebe.engine.state.migration - Skipping ProcessMessageSubscriptionSentTimeMigration migration (1/2).  It was determined it does not need to run right now.
2022-02-02 22:27:49.885 [Broker-0-StreamProcessor-1] [Broker-0-zb-actors-0] INFO  io.camunda.zeebe.engine.state.migration - Skipping MessageSubscriptionSentTimeMigration migration (2/2).  It was determined it does not need to run right now.
2022-02-02 22:27:49.885 [Broker-0-StreamProcessor-1] [Broker-0-zb-actors-0] INFO  io.camunda.zeebe.engine.state.migration - Completed processing of migration tasks (use LogLevel.DEBUG for more details) ... 
2022-02-02 22:27:49.898 [Broker-0-ZeebePartition-1] [Broker-0-zb-actors-1] INFO  io.camunda.zeebe.broker.system - Transition to LEADER on term 2 completed
2022-02-02 22:27:49.930 [Broker-0-HealthCheckService] [Broker-0-zb-actors-1] INFO io.camunda.zeebe.broker.system - All partitions are installed. Broker is ready!
2022-02-02 22:27:49.960 [] [main] INFO io.camunda.zeebe.broker.system - Bootstrap Broker-0 [9/10]: register diskspace usage listeners
2022-02-02 22:27:49.961 [] [main] INFO  io.camunda.zeebe.broker.system - Bootstrap Broker-0 [10/10]: upgrade manager
2022-02-02 22:27:49.966 [] [main] INFO  io.camunda.zeebe.broker.system - Bootstrap Broker-0 succeeded. Started 10 steps in 3352 ms.
2022-02-02 22:27:50.313 [Broker-0-Exporter-1] [Broker-0-zb-fs-workers-1] INFO  io.camunda.zeebe.broker.exporter.elasticsearch - Exporter opened
```
### (7). 总结
在这里仅仅是对Zeebe Broker进行安装,下一小篇会通过Client(Job Workers)与Broker进行通信.   