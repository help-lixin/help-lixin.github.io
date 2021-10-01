---
layout: post
title: 'Kafka集群安装(二)' 
date: 2021-10-01
author: 李新
tags:  Kafka
---

### (1). 概述
在这一篇主要是对Kakfa进行集群的安装.

### (2). 环境准备
+ Jdk1.8(略) 
+ Zookeeper(略) 
+ kafka_2.13-2.8.1

### (3). 机器准备
> 由于我机器有限,所以,在本地机器上创建:伪集群.  

| broker.id    |  机器IP       | 端口  |
|  ----        |  ----        | ----  |
| 0            | 127.0.0.1    | 9092  |
| 1            | 127.0.0.1    | 9093  |
| 2            | 127.0.0.1    | 9094  |

### (4). 下载并解压kafka
```
# 1. 查看当前所在的目录
lixin-macbook:Developer lixin$ pwd
/Users/lixin/Developer

# 2. 创建kafka应用程序目录
lixin-macbook:Developer lixin$ mkdir kafka-2.8.1

# 3. 下载kafka(2.8.1)
lixin-macbook:Developer lixin$ wget https://archive.apache.org/dist/kafka/2.8.1/kafka_2.13-2.8.1.tgz


# 4. 解压并移动到应用程序目录下,同时,创建kafka数据目录
# 解压
lixin-macbook:Developer lixin$ tar -zxvf kafka_2.13-2.8.1.tgz
# 移动解压后的应用到kafka-2.8.1目录下,并重命名为:kafka-0
lixin-macbook:Developer lixin$ mv kafka_2.13-2.8.1 ./kafka-2.8.1/kafka_0
# 为kafka broker创建数据目录
lixin-macbook:Developer lixin$ mkdir -p  ./kafka-2.8.1/kafka_0/data/kafka-logs


# 5. 创建其它kafka brokder
lixin-macbook:Developer lixin$ cp -rf kafka-2.8.1/kafka_0 kafka-2.8.1/kafka_1
lixin-macbook:Developer lixin$ cp -rf kafka-2.8.1/kafka_0 kafka-2.8.1/kafka_2

# 6. 查看应用程序最后的目录
lixin-macbook:Developer lixin$ tree kafka-2.8.1 -L 2
kafka-2.8.1
├── kafka_0
│   ├── LICENSE
│   ├── NOTICE
│   ├── bin
│   ├── config
│   ├── data
│   ├── libs
│   ├── licenses
│   └── site-docs
├── kafka_1
│   ├── LICENSE
│   ├── NOTICE
│   ├── bin
│   ├── config
│   ├── data
│   ├── libs
│   ├── licenses
│   └── site-docs
└── kafka_2
    ├── LICENSE
    ├── NOTICE
    ├── bin
    ├── config
    ├── data
    ├── libs
    ├── licenses
    └── site-docs
```
### (5). 配置kafka(server.properties)

+ kafka_0配置(server.properties)

```
# vi kafka-2.8.1/kafka_0/config/server.properties
broker.id=0
listeners=PLAINTEXT://127.0.0.1:9092
log.dirs=/Users/lixin/Developer/kafka-2.8.1/kafka_0/data/kafka-logs
zookeeper.connect=127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183
```

+ kafka_1配置(server.properties)

```
# vi kafka-2.8.1/kafka_1/config/server.properties
broker.id=1
listeners=PLAINTEXT://127.0.0.1:9093
log.dirs=/Users/lixin/Developer/kafka-2.8.1/kafka_1/data/kafka-logs
zookeeper.connect=127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183
```

+ kafka_2配置(server.properties)

```
# vi kafka-2.8.1/kafka_2/config/server.properties
broker.id=2
listeners=PLAINTEXT://127.0.0.1:9094
log.dirs=/Users/lixin/Developer/kafka-2.8.1/kafka_2/data/kafka-logs
zookeeper.connect=127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183
```
### (6). 启动zk(略)

### (7). 创建一键启动kafka集群脚本(/kafka-2.8.1/start-all.sh)
```
#!/bin/bash
/Users/lixin/Developer/kafka-2.8.1/kafka_0/bin/kafka-server-start.sh   -daemon /Users/lixin/Developer/kafka-2.8.1/kafka_0/config/server.properties
/Users/lixin/Developer/kafka-2.8.1/kafka_1/bin/kafka-server-start.sh   -daemon /Users/lixin/Developer/kafka-2.8.1/kafka_1/config/server.properties
/Users/lixin/Developer/kafka-2.8.1/kafka_2/bin/kafka-server-start.sh   -daemon /Users/lixin/Developer/kafka-2.8.1/kafka_2/config/server.properties
```
### (8). 启动kafka
```
# 1. 启动kafka
lixin-macbook:Developer lixin$ ./kafka-2.8.1/start-all.sh

# 2. 验证:9092/9093/9094
lixin-macbook:Developer lixin$ lsof -i tcp:9092
COMMAND  PID  USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
java    4349 lixin  131u  IPv6 0xa1e662822cae0b7d      0t0  TCP localhost:XmlIpcRegSvc (LISTEN)
java    4349 lixin  147u  IPv6 0xa1e662822d8faebd      0t0  TCP localhost:50471->localhost:XmlIpcRegSvc (ESTABLISHED)
java    4349 lixin  151u  IPv6 0xa1e662822d8fc1dd      0t0  TCP localhost:XmlIpcRegSvc->localhost:50471 (ESTABLISHED)

lixin-macbook:Developer lixin$ lsof -i tcp:9093
COMMAND  PID  USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
java    4349 lixin  155u  IPv6 0xa1e662822d8b651d      0t0  TCP localhost:50472->localhost:9093 (ESTABLISHED)
java    4672 lixin  131u  IPv6 0xa1e662822d91651d      0t0  TCP localhost:9093 (LISTEN)
java    4672 lixin  147u  IPv6 0xa1e662822d915ebd      0t0  TCP localhost:9093->localhost:50472 (ESTABLISHED)

lixin-macbook:Developer lixin$ lsof -i tcp:9094
COMMAND  PID  USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
java    4349 lixin  159u  IPv6 0xa1e662822d9171dd      0t0  TCP localhost:50473->localhost:9094 (ESTABLISHED)
java    4995 lixin  131u  IPv6 0xa1e662822d916b7d      0t0  TCP localhost:9094 (LISTEN)
java    4995 lixin  147u  IPv6 0xa1e662822db1e51d      0t0  TCP localhost:9094->localhost:50473 (ESTABLISHED)
```
### (9). 查看并验证zk内容
```
# 1. 查看有多少brokder往zk里注册
[zk: localhost:2181(CONNECTED) 8] ls /brokers/ids
[0, 1, 2]

# 2. 查看第一个brokder-0
[zk: localhost:2181(CONNECTED) 9] get /brokers/ids/0
{"features":{},"listener_security_protocol_map":{"PLAINTEXT":"PLAINTEXT"},"endpoints":["PLAINTEXT://127.0.0.1:9092"],"jmx_port":-1,"port":9092,"host":"127.0.0.1","version":5,"timestamp":"1633056051250"}

# 3. 查看第一个brokder-1
[zk: localhost:2181(CONNECTED) 10] get /brokers/ids/1
{"features":{},"listener_security_protocol_map":{"PLAINTEXT":"PLAINTEXT"},"endpoints":["PLAINTEXT://127.0.0.1:9093"],"jmx_port":-1,"port":9093,"host":"127.0.0.1","version":5,"timestamp":"1633056052756"}

# 4. 查看第一个brokder-2
[zk: localhost:2181(CONNECTED) 11] get /brokers/ids/2
{"features":{},"listener_security_protocol_map":{"PLAINTEXT":"PLAINTEXT"},"endpoints":["PLAINTEXT://127.0.0.1:9094"],"jmx_port":-1,"port":9094,"host":"127.0.0.1","version":5,"timestamp":"1633056053811"}
```
### (10). 查看kafka集群的管理者
```
[zk: localhost:2181(CONNECTED) 34] get /controller
{"version":1,"brokerid":0,"timestamp":"1633056051564"}
```
### (11). 总结
Kafka集群搭建还是比较简单的,后面对Kafka的一些基本命令进行学习.    