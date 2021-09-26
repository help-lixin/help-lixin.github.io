---
layout: post
title: 'Debezium本地运行(三)' 
date: 2021-09-25
author: 李新
tags:  Debezium
---

### (1). 概述
前面通过Docker运行了Debezium,但是,通过Docker之后,你会感觉到Debezium是一个黑盒了,所以,从这一篇开始,
通过源码编译Debezium,并,在本机跑Debezium的Demo. 

### (2). 准备工作
+ MySQL开启binlog.  
+ 运行Zookeeper.
+ 运行Kafka Brokder.
+ 运行Kafka Connect.
+ 运行Kafka Consumer. 

### (3). MySQL 配置
+ 配置my.cnf

```
log-bin=mysql-bin
binlog_format=row
server-id=1
```

+ 查看binlog位置

```
mysql> SHOW MASTER STATUS \G
*************************** 1. row ***************************
             File: mysql-bin.000393
         Position: 154
     Binlog_Do_DB:
 Binlog_Ignore_DB:
Executed_Gtid_Set:
1 row in set (0.00 sec)
```

+ 创建同步账号和密码

```
GRANT SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'debezium' IDENTIFIED BY 'dbz';
```
### (4). 运行Zookeeper(略)

### (5). Kafka单机搭建(略)

### (6). 配置Kafka Connector
```
# 1. 进入kafka安装目录
lixin-macbook:~ lixin$ cd ~/Developer/kafka-single/

# 2. 查看目录信息
lixin-macbook:kafka-single lixin$ ll
drwxr-xr-x@  39 lixin  staff   1248  9 14 21:09 bin/
drwxr-xr-x@  18 lixin  staff    576  9 25 11:29 config/
drwxr-xr-x    2 lixin  staff     64  9 25 11:26 data/
drwxr-xr-x@ 105 lixin  staff   3360  9 14 21:09 libs/
drwxr-xr-x@  12 lixin  staff    384  9 14 21:09 licenses/
drwxr-xr-x@   3 lixin  staff     96  9 14 21:09 site-docs/

# 3. 创建kakfa-connector目录
lixin-macbook:kafka-single lixin$ mkdir connect
lixin-macbook:kafka-single lixin$ cd connect/

# 4. 拷贝:debezium-connector-mysql到kafka目录下(connect)
lixin-macbook:connect lixin$ cp /Users/lixin/Developer/debezium/debezium-connector-mysql/target/debezium-connector-mysql-1.7.0-SNAPSHOT-plugin.tar.gz  ./
# 4.1 解压插件
lixin-macbook:connect lixin$ tar -zxvf debezium-connector-mysql-1.7.0-SNAPSHOT-plugin.tar.gz
# 4.2 删除压缩文件
lixin-macbook:connect lixin$ rm -rf debezium-connector-mysql-1.7.0-SNAPSHOT-plugin.tar.gz
# 4.3 查看connect目录下的内容
lixin-macbook:connect lixin$ ll
drwxr-xr-x  19 lixin  staff  608  9 25 11:36 debezium-connector-mysql/

# 5. 查看:connect目录结构.
lixin-macbook:connect lixin$ pwd
/Users/lixin/Developer/kafka-single/connect

# *************************************************************************
# 注意:我的mysql是8.0以上的
# *************************************************************************
lixin-macbook:connect lixin$ tree
.
└── debezium-connector-mysql
    ├── antlr4-runtime-4.8.jar
    ├── debezium-api-1.7.0-SNAPSHOT.jar
    ├── debezium-connector-mysql-1.7.0-SNAPSHOT.jar
    ├── debezium-core-1.7.0-SNAPSHOT.jar
    ├── debezium-ddl-parser-1.7.0-SNAPSHOT.jar
    ├── failureaccess-1.0.1.jar
    ├── guava-30.0-jre.jar
    ├── mysql-binlog-connector-java-0.25.1.jar
    └── mysql-connector-java-8.0.26.jar
```
### (7). 启动Kafka Broder
```
lixin-macbook:kafka-single lixin$ ./bin/kafka-server-start.sh -daemon ./config/server.properties
```
### (8). 启动Kafka connector集群模式
+ 配置connect-distributed.properties

```
# (connect-distributed.properties)增加如下内容即可
rest.host.name=127.0.0.1
rest.port=8083
rest.advertised.host.name=127.0.0.1
rest.advertised.port=8083
plugin.path=/Users/lixin/Developer/kafka-single/external_libs
internal.value.converter=org.apache.kafka.connect.json.JsonConverter
task.shutdown.graceful.timeout.ms=10000
offset.flush.timeout.ms=5000
internal.key.converter=org.apache.kafka.connect.json.JsonConverter
```

+ 启动kakfa-connector

```
# 1. 启动kakfa-connector(指定:connect-distributed.properties)
lixin-macbook:kafka-single lixin$ ./bin/connect-distributed.sh  ./config/connect-distributed.properties

# 2. 检查8083端口是否启动成功
lixin-macbook:~ lixin$ lsof -i tcp:8083
COMMAND  PID  USER   FD   TYPE            DEVICE SIZE/OFF NODE NAME
java    9222 lixin  116u  IPv6 0x297c8de3ac9b05f      0t0  TCP localhost:us-srv (LISTEN)
```

+ 检查connector运行情况

```
# 查看connect信息
lixin-macbook:~ lixin$ curl -H "Accept:application/json" localhost:8083/
{"version":"2.8.1","commit":"839b886f9b732b15","kafka_cluster_id":"EHsmpSxxRteUElGdbyDE9A"}

# 查看所有的connectors
lixin-macbook:~ lixin$ curl -H "Accept:application/json" localhost:8083/connectors/
[]
```

+ 向kafka connector注册(MySQL connector)

```
# 向kafka connector注册(mysql connector信息)
# 给自己挖了个坑,我在mac下把mysql的端口改成了3307,然后,debezium连接的信息配置的端口是:3306,但是,报错信息却是这样:  
#   The last packet sent successfully to the server was 0 milliseconds ago. The driver has not received any packets from the server.
#    com.mysql.jdbc.exceptions.jdbc4.CommunicationsException: Communications link failure
#   The last packet sent successfully to the server was 0 milliseconds ago. The driver has not received any packets from the server.
# 注意:mysql8.0.26的driver是支持mysql5.7数据库的.  

lixin-macbook:~ lixin$ curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d '{ "name": "test-connector", "config": { "connector.class": "io.debezium.connector.mysql.MySqlConnector", "tasks.max": "1", "database.hostname": "127.0.0.1", "database.port": "3306", "database.user": "debezium", "database.password": "dbz", "database.server.id": "1", "database.server.name": "test2", "database.whitelist": "test2", "database.history.kafka.bootstrap.servers": "127.0.0.1:9092", "database.history.kafka.topic": "dbhistory.test2" } }'
```
### (9). 查看kafka目录信息
```
# 1. 查看有哪些表
mysql> show tables;
+-----------------+
| Tables_in_test2 |
+-----------------+
| test            |
| user            |
+-----------------+

# 2. 查看kafka数据目上录
lixin-macbook:~ lixin$ ll ~/Developer/kafka-single/data/

# 3. 存放的是数据库的信息(binlog/position)
drwxr-xr-x   6 lixin  staff   192  9 26 21:44 dbhistory.test2-0/             # binlog信息

# 4. 存放的是表结构信息和表里的数据
drwxr-xr-x   6 lixin  staff   192  9 26 21:44 test2-0/                       # 表结构信息
drwxr-xr-x   6 lixin  staff   192  9 26 21:44 test2.test2.test-0/            # 表里的数据
drwxr-xr-x   6 lixin  staff   192  9 26 21:44 test2.test2.user-0/            # 表里的数据
```
### (10). 查看kafka topic列表
```
lixin-macbook:kafka-single lixin$ ./bin/kafka-topics.sh --list --zookeeper 127.0.0.1:2181
__consumer_offsets
dbhistory.test2                               # 存binlog的相关信息.
test2                                         # 数据库的基本信息
test2.test2.test                              # 表里的数据信息
test2.test2.user                              # 表里的数据信息
```
### (11). 通过kafka进行消费  
```
# 1. 监听队列消费
lixin-macbook:kafka-single lixin$ ./bin/kafka-console-consumer.sh --bootstrap-server 127.0.0.1:9092  --from-beginning --topic test2.test2.user

# 2. mysql修改数据
mysql> INSERT INTO user(id,nick,phone,password,email,account) VALUES(7,'小娜',999999,999999,'test@126.com',NULL);

# 3. kafka日志
{"schema":{"type":"struct","fields":[{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":true,"field":"nick"},{"type":"string","optional":true,"field":"phone"},{"type":"string","optional":true,"field":"password"},{"type":"string","optional":true,"field":"email"},{"type":"string","optional":true,"field":"account"}],"optional":true,"name":"test2.test2.user.Value","field":"before"},{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":true,"field":"nick"},{"type":"string","optional":true,"field":"phone"},{"type":"string","optional":true,"field":"password"},{"type":"string","optional":true,"field":"email"},{"type":"string","optional":true,"field":"account"}],"optional":true,"name":"test2.test2.user.Value","field":"after"},{"type":"struct","fields":[{"type":"string","optional":false,"field":"version"},{"type":"string","optional":false,"field":"connector"},{"type":"string","optional":false,"field":"name"},{"type":"int64","optional":false,"field":"ts_ms"},{"type":"string","optional":true,"name":"io.debezium.data.Enum","version":1,"parameters":{"allowed":"true,last,false"},"default":"false","field":"snapshot"},{"type":"string","optional":false,"field":"db"},{"type":"string","optional":true,"field":"sequence"},{"type":"string","optional":true,"field":"table"},{"type":"int64","optional":false,"field":"server_id"},{"type":"string","optional":true,"field":"gtid"},{"type":"string","optional":false,"field":"file"},{"type":"int64","optional":false,"field":"pos"},{"type":"int32","optional":false,"field":"row"},{"type":"int64","optional":true,"field":"thread"},{"type":"string","optional":true,"field":"query"}],"optional":false,"name":"io.debezium.connector.mysql.Source","field":"source"},{"type":"string","optional":false,"field":"op"},{"type":"int64","optional":true,"field":"ts_ms"},{"type":"struct","fields":[{"type":"string","optional":false,"field":"id"},{"type":"int64","optional":false,"field":"total_order"},{"type":"int64","optional":false,"field":"data_collection_order"}],"optional":true,"field":"transaction"}],"optional":false,"name":"test2.test2.user.Envelope"},"payload":{"before":null,"after":{"id":7,"nick":"小娜","phone":"999999","password":"999999","email":"test@126.com","account":null},"source":{"version":"1.6.1.Final","connector":"mysql","name":"test2","ts_ms":1632664785000,"snapshot":"false","db":"test2","sequence":null,"table":"user","server_id":1,"gtid":null,"file":"mysql-bin.000405","pos":923,"row":0,"thread":null,"query":null},"op":"c","ts_ms":1632664785925,"transaction":null}}
```
### (12). 总结
通过本地部署一套应用后,后面会对Debezinum源码进行深度剖析.  