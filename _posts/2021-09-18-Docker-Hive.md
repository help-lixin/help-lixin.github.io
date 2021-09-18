---
layout: post
title: 'Doker运行Hive' 
date: 2021-09-18
author: 李新
tags:  Hive
---


### (1). 前言
自己搭建一台Hive,需要很长的时间(依赖Hadoop),现在Docker的流行,在网上找了一个,发现,Hive有对Docker的支持,所以,只是为了测试,就偷个懒.    

### (2). 下载docker-compose并运行
```
lixin@lixin ~ % cd ~/GitWorkspace
lixin@lixin GitWorkspace % git clone https://github.com/help-lixin/docker-hive.git
lixin@lixin GitWorkspace % cd docker-hive
lixin@lixin docker-hive %  docker-compose up -d
```
### (3). 查看运行docker
```
lixin@lixin docker-hive % docker-compose ps
                 Name                                Command                  State                                  Ports
------------------------------------------------------------------------------------------------------------------------------------------------------
docker-hive_datanode_1                    /entrypoint.sh /run.sh           Up (healthy)   0.0.0.0:50075->50075/tcp,:::50075->50075/tcp
docker-hive_hive-metastore-postgresql_1   /docker-entrypoint.sh postgres   Up             5432/tcp
docker-hive_hive-metastore_1              entrypoint.sh /opt/hive/bi ...   Up             10000/tcp, 10002/tcp,
                                                                                          0.0.0.0:9083->9083/tcp,:::9083->9083/tcp
docker-hive_hive-server_1                 entrypoint.sh /bin/sh -c s ...   Up             0.0.0.0:10000->10000/tcp,:::10000->10000/tcp, 10002/tcp
docker-hive_namenode_1                    /entrypoint.sh /run.sh           Up (healthy)   0.0.0.0:50070->50070/tcp,:::50070->50070/tcp
docker-hive_presto-coordinator_1          ./bin/launcher run               Up             0.0.0.0:8080->8080/tcp,:::8080->8080/tcp
```
### (4). 进行docker容器内部
```
# 1.进入docker容器内部
lixin@lixin docker-hive % docker-compose exec hive-server bash
# 2. 使用beeline客户端连接
root@9fb6f2cf18e5:/opt# /opt/hive/bin/beeline -u jdbc:hive2://localhost:10000
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/opt/hive/lib/log4j-slf4j-impl-2.6.2.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/opt/hadoop-2.7.4/share/hadoop/common/lib/slf4j-log4j12-1.7.10.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
SLF4J: Actual binding is of type [org.apache.logging.slf4j.Log4jLoggerFactory]
Connecting to jdbc:hive2://localhost:10000
Connected to: Apache Hive (version 2.3.2)
Driver: Hive JDBC (version 2.3.2)
Transaction isolation: TRANSACTION_REPEATABLE_READ
Beeline version 2.3.2 by Apache Hive
# 3. 创建表
0: jdbc:hive2://localhost:10000> CREATE TABLE pokes (foo INT, bar STRING);

# 4. 加载磁盘上数据到表里
0: jdbc:hive2://localhost:10000> LOAD DATA LOCAL INPATH '/opt/hive/examples/files/kv1.txt' OVERWRITE INTO TABLE pokes;

# 5. 查询数据
0: jdbc:hive2://localhost:10000> SELECT * FROM pokes;
+------------+------------+
| pokes.foo  | pokes.bar  |
+------------+------------+
| 238        | val_238    |
| 86         | val_86     |
| 311        | val_311    |
+------------+------------+
```
### (5). 总结
Docker能有效的解析环境问题,有时为了测试,而无须自己手动安装一整套Hadoop.   