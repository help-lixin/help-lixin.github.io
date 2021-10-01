---
layout: post
title: 'Kafka常用命令(三)' 
date: 2021-10-01
author: 李新
tags:  Kafka
---

### (1). 概述
前面自己搭了一个Kafka集群,在这里对Kafka的常用命令进行一个学习

### (2). 创建主题
```
# 1. 创建主题,实际上是往zk进行注册,然后,kafka内部监听ZK,去实现主题和分区的创建.

# --create                    --> 创建主题
# --zookeeper 127.0.0.1:2181  --> zk地址
# --partitions 2              --> 指定创建的主题的分区个数(hello-0/hello-1)
# --replication-factor 3      --> 主题对应的副本数()
lixin-macbook:Developer lixin$ ./kafka-2.8.1/kafka_0/bin/kafka-topics.sh --create --zookeeper 127.0.0.1:2181 --replication-factor 3 --partitions 2 --topic hello
Created topic hello.


# 2. 查看创建的主题在磁盘上的信息.
lixin-macbook:Developer lixin$ tree ./kafka-2.8.1/kafka_0/data/kafka-logs/
./kafka-2.8.1/kafka_0/data/kafka-logs/
├── cleaner-offset-checkpoint
├── hello-0
│   ├── 00000000000000000000.index
│   ├── 00000000000000000000.log
│   ├── 00000000000000000000.timeindex
│   └── leader-epoch-checkpoint
├── hello-1
│   ├── 00000000000000000000.index
│   ├── 00000000000000000000.log
│   ├── 00000000000000000000.timeindex
│   └── leader-epoch-checkpoint
├── log-start-offset-checkpoint
├── meta.properties
├── recovery-point-offset-checkpoint
└── replication-offset-checkpoint

lixin-macbook:Developer lixin$ tree ./kafka-2.8.1/kafka_1/data/kafka-logs/
./kafka-2.8.1/kafka_1/data/kafka-logs/
├── cleaner-offset-checkpoint
├── hello-0
│   ├── 00000000000000000000.index
│   ├── 00000000000000000000.log
│   ├── 00000000000000000000.timeindex
│   └── leader-epoch-checkpoint
├── hello-1
│   ├── 00000000000000000000.index
│   ├── 00000000000000000000.log
│   ├── 00000000000000000000.timeindex
│   └── leader-epoch-checkpoint
├── log-start-offset-checkpoint
├── meta.properties
├── recovery-point-offset-checkpoint
└── replication-offset-checkpoint

lixin-macbook:Developer lixin$ tree ./kafka-2.8.1/kafka_2/data/kafka-logs/
./kafka-2.8.1/kafka_2/data/kafka-logs/
├── cleaner-offset-checkpoint
├── hello-0
│   ├── 00000000000000000000.index
│   ├── 00000000000000000000.log
│   ├── 00000000000000000000.timeindex
│   └── leader-epoch-checkpoint
├── hello-1
│   ├── 00000000000000000000.index
│   ├── 00000000000000000000.log
│   ├── 00000000000000000000.timeindex
│   └── leader-epoch-checkpoint
├── log-start-offset-checkpoint
├── meta.properties
├── recovery-point-offset-checkpoint
└── replication-offset-checkpoint


# 3. 通过ZK,查看有哪些分区
[zk: localhost:2181(CONNECTED) 4] ls /brokers/topics/hello/partitions
[0, 1]

# 4. 通过ZK,查看leader信息以及isr
[zk: localhost:2181(CONNECTED) 7] get /brokers/topics/hello/partitions/0/state
{"controller_epoch":3,"leader":2,"version":1,"leader_epoch":0,"isr":[2,0,1]}
```
### (3). 列出所有的主题
```
# 直接查找ZK中的数据就好了
lixin-macbook:Developer lixin$ ./kafka-2.8.1/kafka_0/bin/kafka-topics.sh --list --zookeeper localhost:2181
hello
```
### (4). 查看主题的详细
```
# --describe               --> 查看主题信息
# Leader                   --> 主题下的分区属于哪个broker进行管理(读写数据)
# Replicas: 2,0,1          --> 主题下的分区在哪些brokder有(冗余,仅同步数据到本地)
# Isr                      --> Isr代表:可以进行数据同步或者已同步的brokder节点,并且,当Leader故障时,会从Isr(brokder)列表里进行选择一个成为Leader
lixin-macbook:Developer lixin$ ./kafka-2.8.1/kafka_0/bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic hello
Topic: hello	TopicId: 56XQ5CJ2SyS3oVVtIz5i8w	PartitionCount: 2	ReplicationFactor: 3	Configs:
	Topic: hello	Partition: 0	Leader: 2	Replicas: 2,0,1	Isr: 2,0,1
	Topic: hello	Partition: 1	Leader: 0	Replicas: 0,1,2	Isr: 0,1,2
```
### (5). 通过Kafka控制台进行消息发送
```
# 指定brokder列表
lixin-macbook:Developer lixin$ ./kafka-2.8.1/kafka_0/bin/kafka-console-producer.sh --broker-list 127.0.0.1:9092,127.0.0.1:9093,127.0.0.1:9094 --topic hello
>hello
>world
```
### (6). 通过Kafka控制台进行消息订阅
```
# --from-beginning                                   --> 指定从头开始消费.
# --consumer-property group.id=testGroupa            --> 指定消费组
lixin-macbook:Developer lixin$ ./kafka-2.8.1/kafka_0/bin/kafka-console-consumer.sh --bootstrap-server 127.0.0.1:9092,127.0.0.1:9093,127.0.0.1:9094 --from-beginning --consumer-property group.id=testGroupa --topic hello
world
hello
```
