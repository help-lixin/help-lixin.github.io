---
layout: post
title: 'Pulsar基本概念(一)' 
date: 2021-10-03
author: 李新
tags:  Pulsar
---

### (1). Pulsar是什么
Pulsar是一个用于服务器到服务器的消息系统,具有多租户、高性能等优势.Pulsar最初由Yahoo开发,目前由Apache软件基金会管理.Pulsar 的关键特性如下:    
+ Pulsar的单个实例原生支持多个集群,可跨机房在集群间无缝地完成消息复制.   
+ 极低的发布延迟和端到端延迟.  
+ 无缝扩展到超过一百万个topic.   
+ 简单的客户端 API,支持 Java、Go、Python 和 C++   
+ 支持多种topic订阅模式(独占订阅、共享订阅、故障转移订阅)   
+ 通过Apache BookKeeper提供的持久化消息存储机制保证消息传递.   
  - 由轻量级的serverless计算框架Pulsar Functions实现流原生的数据处理.   
  - 基于Pulsar Functions的serverless connector框架Pulsar IO使得数据更易移入、移出 Apache Pulsar.  
  - 分层式存储可在数据陈旧时,将数据从热存储卸载到冷/长期存储(如S3、GCS)中.  

### (2). Pulsar架构图
!["Pulsar架构图"](/assets/pulsar/imgs/pulsar-system-architecture.png)

### (3). Pulsar架构详解
+ Brokers
  - Pulsar的Broker是一个无状态组件,主要负责运行另外的两个组件(REST API/调度分发器)
+ Apache ZooKeeper
  - Pulsar使用Apache ZooKeeper进行元数据存储、集群配置和协调
+ Apache BookKeeper
  - Pulsar用Apache BookKeeper作为持久化存储.BookKeeper是一个分布式的预写日志(WAL)系统.   
+ Pulsar Proxy
  - Pulsar客户端和Pulsar集群交互的一种方式就是直连Pulsar brokers.然而,在某些情况下,这种直连既不可行也不可取,因为客户端并不知道broker的地址.例如:云环境.   

### (4). Tenant(租户)
> Pulsar天然支持多租户,目的是为了让多用户环境下使用同一套程序,且保证用户间数据隔离,你可以把他理解成一个Windows系统,可以创建不同的User,多User共用一套Windows环境.  

```
# 列出所有的租户信息
lixin-macbook:bin lixin$ ./pulsar-admin tenants list
"public"
"pulsar"
"sample"

# 创建租户: 00007
lixin-macbook:bin lixin$ ./pulsar-admin tenants create 00007

# 查看租户: 00007信息 
lixin-macbook:bin lixin$ ./pulsar-admin tenants get  00007
{
  "adminRoles" : [ ],
  "allowedClusters" : [ "standalone" ]
}

# 更新租户配置信息
lixin-macbook:bin lixin$ ./pulsar-admin tenants update -r dev,test 00007

# 查看租户: 00007角色信息
lixin-macbook:bin lixin$ ./pulsar-admin tenants get  00007
{
  "adminRoles" : [ "dev", "test" ],
  "allowedClusters" : [ "standalone" ]
}
```
### (5). Namespace(命名空间)
> 在Pulsar里对Namesapce这一层级,可以设置权限,调整副本,管理跨集群消息复制等,一个Topic可以继承其所对应namespace里的所有属性,因为,我们只需要对namespace属性进行设置,其下所有的topic都会继承其基本属性.  

```
# 创建namespace(命名空间)
lixin-macbook:bin lixin$ ./pulsar-admin namespaces create 00007/default

# 查看租户下所有的命名空间信息
lixin-macbook:bin lixin$ ./pulsar-admin namespaces list 00007
"00007/default"

# 删除00007租户下的命名空间:default
lixin-macbook:bin lixin$ ./pulsar-admin namespaces delete  00007/default

# 查看namespace的策略信息
lixin-macbook:bin lixin$ ./pulsar-admin namespaces policies 00007/default
{
  "auth_policies" : {
    "namespace_auth" : { },
    "destination_auth" : { },
    "subscription_auth_roles" : { }
  },
  "replication_clusters" : [ "standalone" ],
  "bundles" : {
    "boundaries" : [ "0x00000000", "0x40000000", "0x80000000", "0xc0000000", "0xffffffff" ],
    "numBundles" : 4
  },
  "backlog_quota_map" : { },
  "clusterDispatchRate" : { },
  "topicDispatchRate" : { },
  "subscriptionDispatchRate" : { },
  "replicatorDispatchRate" : { },
  "clusterSubscribeRate" : { },
  "publishMaxMessageRate" : { },
  "latency_stats_sample_rate" : { },
  "deleted" : false,
  "encryption_required" : false,
  "subscription_auth_mode" : "None",
  "offload_threshold" : -1,
  "schema_auto_update_compatibility_strategy" : "Full",
  "schema_compatibility_strategy" : "UNDEFINED",
  "is_allow_auto_update_schema" : true,
  "schema_validation_enforced" : false,
  "subscription_types_enabled" : [ ],
  "properties" : { }
}
```
### (6). Topic(主题)
> Topic可以理解成是对数据进行分类管理,可以把不同的消息放到不同的Topic里,同时,在Topic下又可以划分为多个分片,进行分布式的存储操作,每个分片还存在副本操作,保证数据不丢失.  

```
# 注意: Pulsar创建Topic后,如果,没有任操作,60s后会自动删除,因为,Topic认为这个topic可能是不活动的,可以进行配置不自动删除.
# brokerDeleteInactiveTopicsEnabled=false

# 1. 创建一个没有分区的topic(order-down)
lixin-macbook:bin lixin$ ./pulsar-admin topics create persistent://00007/default/order-down-1

# 2. 查看没有分区的主题
lixin-macbook:bin lixin$ ./pulsar-admin topics list 00007/default
"persistent://00007/default/order-down-1"

# 3. 删除没有分区的主题
lixin-macbook:bin lixin$ ./pulsar-admin topics delete persistent://00007/default/order-down-1


# 4. 创建一个有4个分区的topic(order-down)
lixin-macbook:bin lixin$ ./pulsar-admin topics create-partitioned-topic persistent://00007/default/order-down --partitions 4

# 5. 查看topic信息
lixin-macbook:bin lixin$ ./pulsar-admin topics list 00007/default
"persistent://00007/default/order-down-partition-0"
"persistent://00007/default/order-down-partition-1"
"persistent://00007/default/order-down-partition-2"
"persistent://00007/default/order-down-partition-3"

# 6. 修改topic信息,增加分区
lixin-macbook:bin lixin$ ./pulsar-admin topics update-partitioned-topic persistent://00007/default/order-down --partitions 6

# 7. 查看命名空间下所有的主题
lixin-macbook:bin lixin$ ./pulsar-admin topics list 00007/default
"persistent://00007/default/order-down-partition-0"
"persistent://00007/default/order-down-partition-1"
"persistent://00007/default/order-down-partition-4"
"persistent://00007/default/order-down-partition-5"
"persistent://00007/default/order-down-partition-2"
"persistent://00007/default/order-down-partition-3"

# 8. 删除有分区的主题
lixin-macbook:bin lixin$ ./pulsar-admin topics delete-partitioned-topic persistent://00007/default/order-down


# 9.  为Topic授权
# 9.1 查看租户00007有哪些ROLE
lixin-macbook:bin lixin$ ./pulsar-admin tenants get 00007
{
  "adminRoles" : [ "dev", "test" ],
  "allowedClusters" : [ "standalone" ]
}

# 9.2 在租户00007下创建一个Topic(order-down)
lixin-macbook:bin lixin$ ./pulsar-admin topics create-partitioned-topic persistent://00007/default/order-down --partitions 4

# 9.3 为topic(order-down)的dev角色配置生产和消费的权限
lixin-macbook:bin lixin$ pulsar-admin topics grant-permission --actions produce,consume --role dev persistent://00007/default/order-down


# 9.4 查看Topic(order-down)有哪些权限
lixin-macbook:bin lixin$ pulsar-admin topics permissions persistent://00007/default/order-down
"dev    [consume, produce]"

# 9.5 撤销权限
lixin-macbook:bin lixin$ pulsar-admin topics revoke-permission --role dev persistent://00007/default/order-down
```
### (7). 租户/命名空间/主题的关系
!["Pulsar Tenant/Namespace/Topic关系图"](/assets/pulsar/imgs/pulsar-tenant-namespace-topic.png)


### (8). Subscription
> Subscription可以理解为Kafka(RocketMQ)中的ConsumerGroup.  

### (9). Exclusive
> Pulsar只允许有一个Consumer(消费者)加入Subscription,如果,有其他Consumer试图订阅都会报错,可以理解为:一个Subscription里只允许一个消费者,相当于独占消费. 

!["Pulsar Exclusive"](/assets/pulsar/imgs/pulsar-subscription-exclusive.webp)
### (10). Failover
> 允许多个Consumer加入同一个Subscription,如果活跃的Consumer出现异常,就由其它Consumer来接受消息,可以理解为:主备模式.

!["Pulsar Failover"](/assets/pulsar/imgs/pulsar-subscription-failover.webp)
### (11). Shared
> 允许多个Consumer加入同一个Subscription,消息会依次发送给Subscription下不同的Consumer,消息只会被一个Consumer消费,当Consumer连接异常,发送到该Consumer且未发送确认消息(ack)的消息,会被重新发送给其它Consumer.  

!["Pulsar Shared"](/assets/pulsar/imgs/pulsar-subscription-shared.webp)
### (12). Key_Shared 
> 允许多个Consumer加入同一个Subscription,拥有相同Key的消息会发送给同一个Consumer.  

!["Pulsar Key Shared"](/assets/pulsar/imgs/pulsar-subscription-key-shared.webp)
