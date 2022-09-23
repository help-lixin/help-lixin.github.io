---
layout: post
title: 'Pulsar集群搭建(二)' 
date: 2021-10-03
author: 李新
tags:  Pulsar
---

### (1). 机器准备

|  机器名称   | ip            |              备注            |
|  ----      | ----          |              ----           |
| erp-100    | 10.211.55.100 | zookeeper/broker/bookkeeper |
| erp-101    | 10.211.55.101 | zookeeper/broker/bookkeeper |
| erp-102    | 10.211.55.102 | zookeeper/broker/bookkeeper |

### (2). ZK集群搭建(略)

### (3). 下载Pulsar
```
[root@erp-XXX ~]# wget --no-check-certificate https://dlcdn.apache.org/pulsar/pulsar-2.8.1/apache-pulsar-2.8.1-bin.tar.gz
[root@erp-XXX ~]# tar -zxvf apache-pulsar-2.8.1-bin.tar.gz
[root@erp-XXX ~]# mv apache-pulsar-2.8.1 pulsar
```
### (4). 下载Pulsar connectors(略)

connectors有点类似于Kafka里的Connector来着的,用于快速将数据导入和导出Pulsar,在这里我不先不做配置,快速的对Pulsar入门.   

### (5). 初始化Pulsar集群元数据到Zookeeper中
```
# 该命令只能执行一次,也仅只要在一台机器上执行即可
# 1. 进入pulsar目录
[root@erp-XXX ~]# cd pulsar/bin/

# --cluster                    : 集群名称
# --zookeeper                  : ZooKeeper集群连接参数,仅需要包含集群中的一个节点即可,因为,ZK内部会做数据同步
# --configuration-store        : Pulsar实例的配置存储集群(ZooKeeper),和-zookeeper参数一样只需要包含集群中的一个节点即可.  
# --web-service-url            : 集群Web服务的URL:端口,URL必须是一个标准的DNS名称,默认端口8080,不建议修改.
# --web-service-url-tls        : 集群Web提供TLS服务的URL:端口,端口默认8443,不建议修改.
# --broker-service-url         : 集群brokers服务URL,URL中DNS的名称和Web服务保持一致,URL使用pulsar替代http/http,端口默认6650,不建议修改.  
# --broker-service-url-tls     : 集群brokers提供TLS服务的URL,默认端口6551,不建议修改

# 2. 初始化集群元数据
[root@erp-XXX bin]# ./pulsar initialize-cluster-metadata \
--cluster pulsar-cluster-prod \
--zookeeper 10.211.55.100:2181 \
--configuration-store 10.211.55.100:2181 \
--web-service-url http://10.211.55.100:8080,10.211.55.101:8080,10.211.55.102:8080 \
--web-service-url-tls https://10.211.55.100:8443,10.211.55.101:8443,10.211.55.102:8443 \
--broker-service-url pulsar://10.211.55.100:6650,10.211.55.101:6650,10.211.55.102:6650 \
--broker-service-url-tls pulsar+ssl://10.211.55.100:6651,10.211.55.101:6651,10.211.55.102:6651


# 3. zk验证元数据
[root@erp-XXX bin]# ./zkCli.sh
[zk: localhost:2181(CONNECTED) 0] ls /
[admin, bookies, ledgers, managed-ledgers, namespace, pulsar, stream, zookeeper]
```
### (6). 配置并启动Apache BookKeeper
```
[root@erp-XXX pulsar]# cd /root/pulsar/

# 1. 修改所有BookKeeper的zk地址
[root@erp-XXX pulsar]# sed -i \
-e 's/zkServers=localhost:2181/zkServers=10.211.55.100:2181,10.211.55.101:2181,10.211.55.102:2181/g' \
/root/pulsar/conf/bookkeeper.conf 

# 2. 启动所有的BookKeeper
# 控制台启动bookie
# [root@erp-XXX pulsar]# ./bin/pulsar bookie
# 后台守护进程启动bookie
[root@erp-XXX pulsar]# ./bin/pulsar-daemon start bookie

# 3. 验证是否成功(Bookie sanity test succeeded)
[root@erp-XXX pulsar]# bin/bookkeeper shell bookiesanity
12:22:55.708 [main] INFO  org.apache.bookkeeper.tools.cli.commands.bookie.SanityTestCommand - Bookie sanity test succeeded

# 4. 启动BookKeeper时,会在pulsar(/root/pulsar/)目录下,自动创建数据存储目录
[root@erp-XXX pulsar]# tree /root/pulsar/data/bookkeeper -L 1
/root/pulsar/data/bookkeeper
├── journal
└── ledgers
```
### (7). 配置并启动Pulsar broker
```
[root@erp-XXX ~]# cd /root/pulsar/

# 1. 修改所有brokder配置
[root@erp-XXX pulsar]# sed -i \-e 's/zookeeperServers=/zookeeperServers=10.211.55.100:2181,10.211.55.101:2181,10.211.55.102:2181/g' \
-e 's/configurationStoreServers=/configurationStoreServers=10.211.55.100:2181,10.211.55.101:2181,10.211.55.102:2181/g' \
-e 's/clusterName=/clusterName=pulsar-cluster-prod/g' \
/root/pulsar/conf/broker.conf 

# 2. 启动所有的brokder
[root@erp-XXX pulsar]# ./bin/pulsar-daemon start broker

# 3. 验证查看集群状态
[root@erp-XXX pulsar]# ./bin/pulsar-admin brokers list pulsar-cluster
"erp-101:8080"
"erp-102:8080"
"erp-100:8080"
```
### (8). 测试验证
```
[root@erp-XXX ~]# cd /root/pulsar/

# 1. 修改所有pulsar的client配置
[root@erp-XXX pulsar]# sed -i \
-e 's/webServiceUrl=http:\/\/localhost:8080\//webServiceUrl=http:\/\/10.211.55.100:8080,10.211.55.101:8080,10.211.55.102:8080\//g' \
-e 's/brokerServiceUrl=pulsar:\/\/localhost:6650\//brokerServiceUrl=pulsar:\/\/10.211.55.100:6650,10.211.55.101:6650,10.211.55.101:6650\//g' \
/root/pulsar/conf/client.conf

# 2. 配置命名空间信息
[root@erp-XXX pulsar]# ./bin/pulsar-admin namespaces set-persistence -a 3 -e 3 -w 3 -r 3 public/default
[root@erp-XXX pulsar]# ./bin/pulsar-admin namespaces get-persistence public/default
{
  "bookkeeperEnsemble" : 3,
  "bookkeeperWriteQuorum" : 3,
  "bookkeeperAckQuorum" : 3,
  "managedLedgerMaxMarkDeleteRate" : 3.0
}

# 3. 启动消费者监听
[root@erp-XXX pulsar]# ./bin/pulsar-client consume persistent://public/default/test -n 100 -s "consumer-test" -t "Exclusive"

# 4. 启动生产者发送消息.
[root@erp-XXX pulsar]# ./bin/pulsar-client produce persistent://public/default/test -n 1 -m "Hello Pulsar"
12:54:47.270 [main] INFO  org.apache.pulsar.client.cli.PulsarClientTool - 1 messages successfully produced

# 5. 查看消费者控制台信息
12:54:27.382 [pulsar-client-io-1-1] INFO  org.apache.pulsar.client.impl.ConnectionPool - [[id: 0x37d924b7, L:/10.211.55.101:42990 - R:10.211.55.101/10.211.55.101:6650]] Connected to server
12:54:27.990 [pulsar-client-io-1-1] INFO  org.apache.pulsar.client.impl.ConnectionPool - [[id: 0x2a653810, L:/10.211.55.101:42992 - R:erp-101/10.211.55.101:6650]] Connected to server
12:54:28.020 [pulsar-client-io-1-1] INFO  org.apache.pulsar.client.impl.ConsumerImpl - [persistent://public/default/test][consumer-test] Subscribing to topic on cnx [id: 0x2a653810, L:/10.211.55.101:42992 - R:erp-101/10.211.55.101:6650], consumerId 0
12:54:29.932 [pulsar-client-io-1-1] INFO  org.apache.pulsar.client.impl.ConsumerImpl - [persistent://public/default/test][consumer-test] Subscribed to topic on erp-101/10.211.55.101:6650 -- consumer: 0
12:54:45.319 [pulsar-client-io-1-1] INFO  com.scurrilous.circe.checksum.Crc32cIntChecksum - SSE4.2 CRC32C provider initialized
----- got message -----
key:[null], properties:[], content:Hello Pulsar
12:55:27.371 [pulsar-timer-5-1] INFO  org.apache.pulsar.client.impl.ConsumerStatsRecorderImpl - [persistent://public/default/test] [consumer-test] [1d382] Prefetched messages: 0 --- Consume throughput received: 0.02 msgs/s --- 0.00 Mbit/s --- Ack sent rate: 0.02 ack/s --- Failed messages: 0 --- batch messages: 0 ---Failed acks: 0
```
### (9). 总结
> Pulsar总体来说,集群搭建还是算顺利的,后面,会利用Java测试调用.   