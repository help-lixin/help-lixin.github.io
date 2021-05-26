---
layout: post
title: 'MongoDB 分片集群搭建(三)'
date: 2021-05-24
author: 李新
tags:  MongoDB
---

### (1). 机器准备

|  机器名称          | IP            | 角色                  |
|  ----            | ----           | ----                 |
|  clickhouse-1    | 10.211.55.100  | Mongs/Config(P/S)   |
|  clickhouse-2    | 10.211.55.101  | Replica Set(P/S/A)   |
|  clickhouse-3    | 10.211.55.102  | Replica Set(P/S/A)   |

### (2). Replica Set搭建
> 由于我的机器有限,所以,在clickhouse-2和clickhouse-3搭建Replica Set(Primary/Secondary/Arbiter).  

```
# 1. 查看当前所在目录
[root@clickhouse-2/3 soft]# pwd
/opt/soft

# 2. 创建日志目录/数据目录/配置目录
[root@clickhouse-2/3 soft]# mkdir -p mongodb-linux-x86_64-rhel70-4.4.6/{27017,27018,27019}/data
[root@clickhouse-2/3 soft]# mkdir -p mongodb-linux-x86_64-rhel70-4.4.6/{27017,27018,27019}/logs
[root@clickhouse-2/3 soft]# mkdir -p mongodb-linux-x86_64-rhel70-4.4.6/{27017,27018,27019}/conf

# 3. 解压
[root@clickhouse-2/3 soft]# tar -zxvf mongodb-linux-x86_64-rhel70-4.4.6.tgz
```
### (3). mongod.conf
> 请注意:写有注释的的内容,在不同的Replica Set是需要做替换的.  

```
systemLog:
    destination: file
	# 需要注意这个配置:/27017/27018/27019
    path: "/opt/soft/mongodb-linux-x86_64-rhel70-4.4.6/{xxxx}/logs/mongodb.log"
    logAppend: true
storage:
    # 需要注意这个配置:/27017/27018/27019
    dbPath: "/opt/soft/mongodb-linux-x86_64-rhel70-4.4.6/{xxxx}/data"
    journal:  # 启用持久化日志以确保数据文件保持有效和可恢复.
        enabled: true
processManagement:
    fork: true
	# 需要注意这个配置:/27017/27018/27019
    pidFilePath: "/opt/soft/mongodb-linux-x86_64-rhel70-4.4.6/{xxxx}/logs/mongodb.pid"
net:
    bindIp: 0.0.0.0
    port: {xxxx}   # 需要注意这个配置:/27017/27018/27019

# 副本集的名称(三台机器要保持一持)
replication:
    # 注意:clickhouse-2配置成:shard-rs02
	# 注意:clickhouse-3配置成:shard-rs03
    replSetName: shard-rs02
sharding:
    # shardsvr   : 分片
	# configsvr  : config
    clusterRole: shardsvr
```
### (4). 目录结构如下
```
[root@clickhouse-2/3 soft]# tree mongodb-linux-x86_64-rhel70-4.4.6
mongodb-linux-x86_64-rhel70-4.4.6
├── 27017
│   ├── conf
│   │   └── mongod.conf
│   ├── data
│   └── logs
├── 27018
│   ├── conf
│   │   └── mongod.conf
│   ├── data
│   └── logs
├── 27019
│   ├── conf
│   │   └── mongod.conf
│   ├── data
│   └── logs
├── bin
│   ├── install_compass
│   ├── mongo
│   ├── mongod
│   └── mongos
└── THIRD-PARTY-NOTICES
```
### (5). 设计启动脚本
```
# 1. 设计启动脚本
[root@clickhouse-2/3 soft]#vi mongodb-linux-x86_64-rhel70-4.4.6/start-27017.sh
 #!/bin/bash
 nohup /opt/soft/mongodb-linux-x86_64-rhel70-4.4.6/bin/mongod -f /opt/soft/mongodb-linux-x86_64-rhel70-4.4.6/27017/conf/mongod.conf  > /dev/null 2>&1 &

# 2. 设置成可执行权限
[root@clickhouse-2/3 soft]# chmod 755 mongodb-linux-x86_64-rhel70-4.4.6/start-27017.sh

# 3. 配置27018和27019
[root@clickhouse-2/3 soft]# cp mongodb-linux-x86_64-rhel70-4.4.6/start-27017.sh mongodb-linux-x86_64-rhel70-4.4.6/start-27018.sh
[root@clickhouse-2/3 soft]# cp mongodb-linux-x86_64-rhel70-4.4.6/start-27017.sh mongodb-linux-x86_64-rhel70-4.4.6/start-27019.sh

# 4. 替换脚本内容
[root@clickhouse-2/3 soft]# sed -i 's/27017/27018/'  mongodb-linux-x86_64-rhel70-4.4.6/start-27018.sh
[root@clickhouse-2/3 soft]# sed -i 's/27017/27019/'  mongodb-linux-x86_64-rhel70-4.4.6/start-27019.sh
```
### (6). 启动所有的节点(Primary/Secondary/Arbiter)
```
# 1. 启动所有的节点
[root@clickhouse-2/3 soft]# ./mongodb-linux-x86_64-rhel70-4.4.6/start-27017.sh
[root@clickhouse-2/3 soft]# ./mongodb-linux-x86_64-rhel70-4.4.6/start-27018.sh
[root@clickhouse-2/3 soft]# ./mongodb-linux-x86_64-rhel70-4.4.6/start-27019.sh

# 2. 查看是否启动成功
[root@clickhouse-2/3 soft]# netstat -tlnp|grep 2701
tcp        0      0 0.0.0.0:27017           0.0.0.0:*               LISTEN      11207/mongod
tcp        0      0 0.0.0.0:27018           0.0.0.0:*               LISTEN      11266/mongod
tcp        0      0 0.0.0.0:27019           0.0.0.0:*               LISTEN      11322/mongod
```
### (7). Primary添加成员
```
# 注意:clickhouse-3节点,也要按如下步骤执行(添加本机端口[27017/27018/27019]):

[root@clickhouse-2 soft]# ./mongodb-linux-x86_64-rhel70-4.4.6/bin/mongo --host=10.211.55.101 --port 27017
MongoDB shell version v4.4.6
# 1. Replica Set初始化
> rs.initiate();
{
        "info2" : "no configuration specified. Using a default configuration for the set",
        "me" : "clickhouse-2:27017",
        "ok" : 1
}

# 2. 添加成员Secondary
cluster:PRIMARY> rs.add("10.211.55.101:27018");
{
        "ok" : 1,
        "$clusterTime" : {
                "clusterTime" : Timestamp(1621946480, 1),
                "signature" : {
                        "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
                        "keyId" : NumberLong(0)
                }
        },
        "operationTime" : Timestamp(1621946480, 1)
}

# 3. 添加成员Arbiter
cluster:PRIMARY> rs.addArb("10.211.55.101:27019");
{
        "ok" : 1,
        "$clusterTime" : {
                "clusterTime" : Timestamp(1621946505, 1),
                "signature" : {
                        "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
                        "keyId" : NumberLong(0)
                }
        },
        "operationTime" : Timestamp(1621946505, 1)
}

# ********************************************************
# 4. 修改第一个成员(把host改成ip),否则,在添加Shard时,会抛错.
#    两个副本集(shard-rs02/shard-rs03)都要这样做.
#    否则,添加Shard时,会抛出异常: "errmsg" : "in seed list shard-rs02/10.211.55.101:27017,10.211.55.101:27018,10.211.55.101:27019, host 10.211.55.101:27017 does not belong to replica set shard-rs02
# ********************************************************
var config=rs.config();
config.members[0].host="10.211.55.101:27017";
rs.reconfig(config);


# 5. 查看Replice Set状态
cluster:PRIMARY> rs.conf();
{
        "_id" : "shard-rs02",
        "version" : 3,
        "term" : 1,
        "protocolVersion" : NumberLong(1),
        "writeConcernMajorityJournalDefault" : true,
        "members" : [
                {
                        "_id" : 0,
                        "host" : "10.211.55.101:27017",   # 注意:这里要是IP:PORT
                        "arbiterOnly" : false,
                        "buildIndexes" : true,
                        "hidden" : false,
                        "priority" : 1,
                        "tags" : {

                        },
                        "slaveDelay" : NumberLong(0),
                        "votes" : 1
                },
                {
                        "_id" : 1,
                        "host" : "10.211.55.101:27018",
                        "arbiterOnly" : false,
                        "buildIndexes" : true,
                        "hidden" : false,
                        "priority" : 1,
                        "tags" : {

                        },
                        "slaveDelay" : NumberLong(0),
                        "votes" : 1
                },
                {
                        "_id" : 2,
                        "host" : "10.211.55.101:27019",
                        "arbiterOnly" : true,
                        "buildIndexes" : true,
                        "hidden" : false,
                        "priority" : 0,
                        "tags" : {

                        },
                        "slaveDelay" : NumberLong(0),
                        "votes" : 1
                }
        ],
        "settings" : {
                "chainingAllowed" : true,
                "heartbeatIntervalMillis" : 2000,
                "heartbeatTimeoutSecs" : 10,
                "electionTimeoutMillis" : 10000,
                "catchUpTimeoutMillis" : -1,
                "catchUpTakeoverDelayMillis" : 30000,
                "getLastErrorModes" : {

                },
                "getLastErrorDefaults" : {
                        "w" : 1,
                        "wtimeout" : 0
                },
                "replicaSetId" : ObjectId("60add9a2010f7ed64994c1ca")
        }
}
```
### (8). Secondary/Arbiter允许成为slave
```
# 注意:clickhouse-3也要这样执行.

[root@clickhouse-2/3 soft]# ./mongodb-linux-x86_64-rhel70-4.4.6/bin/mongo --host=10.211.55.101 --port 27018
MongoDB shell version v4.4.6
# 1. 允许成为slave
cluster:SECONDARY> rs.slaveOk();
WARNING: slaveOk() is deprecated and may be removed in the next major release. Please use secondaryOk() instead.

# 2. 查看所有的数据库
cluster:SECONDARY> show dbs;
admin   0.000GB
config  0.000GB
local   0.000GB

# 3. ARBITER允许成为slave
[root@clickhouse-2 soft]# ./mongodb-linux-x86_64-rhel70-4.4.6/bin/mongo --host=10.211.55.101 --port 27019
MongoDB shell version v4.4.6
cluster:ARBITER> rs.slaveOk();
WARNING: slaveOk() is deprecated and may be removed in the next major release. Please use secondaryOk() instead.

# 4. 无法查看dbs(因为ARBITER本来就不存储数据哈)
cluster:ARBITER> show dbs;
uncaught exception: Error: listDatabases failed:{
        "topologyVersion" : {
                "processId" : ObjectId("60aceeec2bf0cea7ba85e90b"),
                "counter" : NumberLong(1)
        },
        "ok" : 0,
        "errmsg" : "node is not in primary or recovering state",
        "code" : 13436,
        "codeName" : "NotPrimaryOrSecondary"
} :
```
### (9). Config搭建
```
# 1. 注意:Config在clickhouse-1上搭建.
[root@clickhouse-1 soft]# pwd
/opt/soft

# 2. 创建数据目录/日志目录/配置目录
[root@clickhouse-1 soft]# mkdir -p mongodb-linux-x86_64-rhel70-4.4.6/{27027,27028,27029}/data
[root@clickhouse-1 soft]# mkdir -p mongodb-linux-x86_64-rhel70-4.4.6/{27027,27028,27029}/logs
[root@clickhouse-1 soft]# mkdir -p mongodb-linux-x86_64-rhel70-4.4.6/{27027,27028,27029}/conf

# 3. 解压
[root@clickhouse-1 soft]# tar -zxvf mongodb-linux-x86_64-rhel70-4.4.6.tgz

# 3. 重命名
[root@clickhouse-1 soft]# mv mongodb-linux-x86_64-rhel70-4.4.6 mongodb-config
```
### (10). Config配置(mongod.conf)  

> 请注意:写有注释的的内容,在不同的Replica Set是需要做替换的.  

```
systemLog:
    destination: file
	# 需要注意这个配置:27027/27028/27029
    path: "/opt/soft/mongodb-config/{xxxx}/logs/mongodb.log"
    logAppend: true
storage:
    # 需要注意这个配置:27027/27028/27029
    dbPath: "/opt/soft/mongodb-config/{xxxx}/data"
    journal:  # 启用持久化日志以确保数据文件保持有效和可恢复.
        enabled: true
processManagement:
    fork: true
	# 需要注意这个配置:27027/27028/27029
    pidFilePath: "/opt/soft/mongodb-config/{xxxx}/logs/mongodb.pid"
net:
    bindIp: 0.0.0.0
    port: {xxxx}  # 需要注意这个配置:27027/27028/27029

# 副本集的名称(三台机器要保持一持)
replication:
    replSetName: config-server
sharding:
    # 
    # shardsvr   : 分片
	# configsvr  : config
    clusterRole: configsvr
```

### (11). Config配置启动脚本
```
# 1. 设计启动脚本
[root@clickhouse-1 soft]# vi mongodb-config/start-27027.sh
#!/bin/bash
nohup /opt/soft/mongodb-config/bin/mongod -f /opt/soft/mongodb-config/27027/conf/mongod.conf  > /dev/null 2>&1 &

# 2. 设置成可执行权限
[root@clickhouse-1 soft]# chmod 755 mongodb-config/start-27027.sh

# 3. 配置27018和27019
[root@clickhouse-1 soft]# cp mongodb-config/start-27027.sh mongodb-config/start-27028.sh
[root@clickhouse-1 soft]# cp mongodb-config/start-27027.sh mongodb-config/start-27029.sh

# 4. 替换脚本内容
[root@clickhouse-1 soft]# sed -i 's/27027/27028/'  mongodb-config/start-27028.sh
[root@clickhouse-1 soft]# sed -i 's/27027/27029/'  mongodb-config/start-27029.sh
```
### (12). Config启动
```
# 1. 启动:27027/27028/27029
[root@clickhouse-1 soft]# ./mongodb-config/start-27027.sh
[root@clickhouse-1 soft]# ./mongodb-config/start-27028.sh
[root@clickhouse-1 soft]# ./mongodb-config/start-27029.sh

# 2. 查看端口是否启动
[root@clickhouse-1 soft]# netstat -tlnp|grep 2702
tcp        0      0 0.0.0.0:27027           0.0.0.0:*               LISTEN      3355/mongod
tcp        0      0 0.0.0.0:27028           0.0.0.0:*               LISTEN      3420/mongod
tcp        0      0 0.0.0.0:27029           0.0.0.0:*               LISTEN      3482/mongod

```
### (13). Config节点初始化
```
[root@clickhouse-1 soft]# ./mongodb-config/bin/mongo --host 127.0.0.1 --port 27027
MongoDB shell version v4.4.6
# 1. 初始化
> rs.initiate();
{
        "info2" : "no configuration specified. Using a default configuration for the set",
        "me" : "clickhouse-1:27027",
        "ok" : 1,
        "$gleStats" : {
                "lastOpTime" : Timestamp(1622010706, 1),
                "electionId" : ObjectId("000000000000000000000000")
        },
        "lastCommittedOpTime" : Timestamp(0, 0)
}

# 2. 添加从节点(27028)
config-server:PRIMARY> rs.add("10.211.55.100:27028");
{
        "ok" : 1,
        "$gleStats" : {
                "lastOpTime" : {
                        "ts" : Timestamp(1622010865, 2),
                        "t" : NumberLong(1)
                },
                "electionId" : ObjectId("7fffffff0000000000000001")
        },
        "lastCommittedOpTime" : Timestamp(1622010865, 2),
        "$clusterTime" : {
                "clusterTime" : Timestamp(1622010867, 1),
                "signature" : {
                        "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
                        "keyId" : NumberLong(0)
                }
        },
        "operationTime" : Timestamp(1622010865, 2)
}

# 3. 添加从节点(27029)
config-server:PRIMARY> rs.add("10.211.55.100:27029");
{
        "ok" : 1,
        "$gleStats" : {
                "lastOpTime" : {
                        "ts" : Timestamp(1622010873, 2),
                        "t" : NumberLong(1)
                },
                "electionId" : ObjectId("7fffffff0000000000000001")
        },
        "lastCommittedOpTime" : Timestamp(1622010875, 1),
        "$clusterTime" : {
                "clusterTime" : Timestamp(1622010875, 1),
                "signature" : {
                        "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
                        "keyId" : NumberLong(0)
                }
        },
        "operationTime" : Timestamp(1622010873, 2)
}

# 4. 查看服务状态
config-server:PRIMARY> rs.conf();
{
        "_id" : "config-server",
        "version" : 3,
        "term" : 1,
        "configsvr" : true,
        "protocolVersion" : NumberLong(1),
        "writeConcernMajorityJournalDefault" : true,
        "members" : [
                {
                        "_id" : 0,
                        "host" : "clickhouse-1:27027",
                        "arbiterOnly" : false,
                        "buildIndexes" : true,
                        "hidden" : false,
                        "priority" : 1,
                        "tags" : {

                        },
                        "slaveDelay" : NumberLong(0),
                        "votes" : 1
                },
                {
                        "_id" : 1,
                        "host" : "10.211.55.100:27028",
                        "arbiterOnly" : false,
                        "buildIndexes" : true,
                        "hidden" : false,
                        "priority" : 1,
                        "tags" : {

                        },
                        "slaveDelay" : NumberLong(0),
                        "votes" : 1
                },
                {
                        "_id" : 2,
                        "host" : "10.211.55.100:27029",
                        "arbiterOnly" : false,
                        "buildIndexes" : true,
                        "hidden" : false,
                        "priority" : 1,
                        "tags" : {

                        },
                        "slaveDelay" : NumberLong(0),
                        "votes" : 1
                }
        ],
        "settings" : {
                "chainingAllowed" : true,
                "heartbeatIntervalMillis" : 2000,
                "heartbeatTimeoutSecs" : 10,
                "electionTimeoutMillis" : 10000,
                "catchUpTimeoutMillis" : -1,
                "catchUpTakeoverDelayMillis" : 30000,
                "getLastErrorModes" : {

                },
                "getLastErrorDefaults" : {
                        "w" : 1,
                        "wtimeout" : 0
                },
                "replicaSetId" : ObjectId("60adeb52fbcf338efdaf8d8a")
        }
}
```
### (14). Mongs创建
```
[root@clickhouse-1 soft]# pwd
/opt/soft

# 2. 创建日志目录/配置目录
[root@clickhouse-1 soft]# mkdir -p mongodb-linux-x86_64-rhel70-4.4.6/{27017,27018}/logs
[root@clickhouse-1 soft]# mkdir -p mongodb-linux-x86_64-rhel70-4.4.6/{27017,27018}/conf

# 3. 解压
[root@clickhouse-1 soft]# tar -zxvf mongodb-linux-x86_64-rhel70-4.4.6.tgz

# 3. 重命名
[root@clickhouse-1 soft]# mv mongodb-linux-x86_64-rhel70-4.4.6 mongos
```
### (15). Mongs配置文件(mongos.conf)
```
systemLog:
    destination: file
	# 需要注意这个配置:/27017/27018
    path: "/opt/soft/mongos/{xxxx}/logs/mongodb.log"
    logAppend: true
processManagement:
    fork: true
	# 需要注意这个配置:/27017/27018
    pidFilePath: "/opt/soft/mongos/{xxxx}/logs/mongodb.pid"
net:
    bindIp: 0.0.0.0
    port: {xxxx}   # 需要注意这个配置:/27017/27018
sharding:
    # 指定配置节点副本集
    configDB:config-server/10.211.55.100:27027,10.211.55.100:27028,10.211.55.100:27029
```
### (16). 配置启动脚本
```
# 1. 设计启动脚本
[root@clickhouse-1 soft]# vi mongos/start-27017.sh
#!/bin/bash
nohup /opt/soft/mongos/bin/mongos -f /opt/soft/mongos/27017/conf/mongos.conf  > /dev/null 2>&1 &

# 2. 设置成可执行权限
[root@clickhouse-1 soft]# chmod 755 mongos/start-27017.sh

# 3. 配置27018
[root@clickhouse-1 soft]# cp mongos/start-27017.sh mongos/start-27018.sh

# 4. 替换脚本内容
[root@clickhouse-1 soft]# sed -i 's/27017/27028/'  mongos/start-27018.sh
```
### (17). 启动mongs
```
# 1. 启动mongs
[root@clickhouse-1 soft]# ./mongos/start-27017.sh

# 2. 检查启动是否成功
[root@clickhouse-1 soft]# lsof -i tcp:27017
COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
mongos  12068 root   12u  IPv4  64952      0t0  TCP *:27017 (LISTEN)
```
### (18). mongs添加Shard
```
[root@clickhouse-1 soft]# ./mongos/bin/mongo --host 10.211.55.100 --port 27017
MongoDB shell version v4.4.6
# 1. 添加第一个副本集(shard-rs02)
mongos> sh.addShard("shard-rs02/10.211.55.101:27017,10.211.55.101:27018,10.211.55.101:27019");
{
        "shardAdded" : "shard-rs02",
        "ok" : 1,
        "operationTime" : Timestamp(1622014710, 7),
        "$clusterTime" : {
                "clusterTime" : Timestamp(1622014710, 7),
                "signature" : {
                        "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
                        "keyId" : NumberLong(0)
                }
        }
}

# 2. 添加第二个副本集(shard-rs03)
mongos> sh.addShard("shard-rs03/10.211.55.102:27017,10.211.55.102:27018,10.211.55.102:27019");
{
        "shardAdded" : "shard-rs03",
        "ok" : 1,
        "operationTime" : Timestamp(1622014757, 6),
        "$clusterTime" : {
                "clusterTime" : Timestamp(1622014757, 6),
                "signature" : {
                        "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
                        "keyId" : NumberLong(0)
                }
        }
}
```
### (19). 启用分片
```
# 1. 对某个库(test)开启分片功能
mongos> sh.enableSharding('test1');
{
        "ok" : 1,
        "operationTime" : Timestamp(1622014940, 6),
        "$clusterTime" : {
                "clusterTime" : Timestamp(1622014940, 6),
                "signature" : {
                        "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
                        "keyId" : NumberLong(0)
                }
        }
}

# 2. 集合分片配置(hash)
#  对test库下的users集合进行分片,分片的规则是根据性别进行hash
mongos> sh.shardCollection("test1.users",{"name":"hashed"});
{
        "collectionsharded" : "test1.users",
        "collectionUUID" : UUID("5f11d2b6-09b2-4c95-9e43-7b18c6221358"),
        "ok" : 1,
        "operationTime" : Timestamp(1622015290, 41),
        "$clusterTime" : {
                "clusterTime" : Timestamp(1622015290, 41),
                "signature" : {
                        "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
                        "keyId" : NumberLong(0)
                }
        }
}

# 3. 集合分片配置(range)
#  对test库下的orders集合进行分片,分片的规则是根据商户id进行范围分片. 
# mongos> sh.shardCollection("test2.orders",{"merchant_id":1});
# {
#        "collectionsharded" : "test2.orders",
#        "collectionUUID" : UUID("ec749771-b613-4c1a-92c2-89863179e69c"),
#        "ok" : 1,
#        "operationTime" : Timestamp(1622015775, 13),
#        "$clusterTime" : {
#                "clusterTime" : Timestamp(1622015775, 13),
#                "signature" : {
#                        "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
#                        "keyId" : NumberLong(0)
#                }
#        }
# }

# 4. 查看分片详细信息
mongos> sh.status();
--- Sharding Status ---
  sharding version: {
        "_id" : 1,
        "minCompatibleVersion" : 5,
        "currentVersion" : 6,
        "clusterId" : ObjectId("60adeb52fbcf338efdaf8d8f")
  }
  shards:
        {  "_id" : "shard-rs02",  "host" : "shard-rs02/10.211.55.101:27017,10.211.55.101:27018",  "state" : 1 }
        {  "_id" : "shard-rs03",  "host" : "shard-rs03/10.211.55.102:27017,10.211.55.102:27018",  "state" : 1 }
  active mongoses:
        "4.4.6" : 1
  autosplit:
        Currently enabled: yes
  balancer:
        Currently enabled:  yes
        Currently running:  no
        Failed balancer rounds in last 5 attempts:  0
        Migration Results for the last 24 hours:
                512 : Success
  databases:
        {  "_id" : "config",  "primary" : "config",  "partitioned" : true }
                config.system.sessions
                        shard key: { "_id" : 1 }
                        unique: false
                        balancing: true
                        chunks:
                                shard-rs02      512
                                shard-rs03      512
                        too many chunks to print, use verbose if you want to force print
        {  "_id" : "test",  "primary" : "shard-rs03",  "partitioned" : true,  "version" : {  "uuid" : UUID("1118b681-706c-4b15-a6c0-a222f5c28e1c"),  "lastMod" : 1 } }
        {  "_id" : "test1",  "primary" : "shard-rs03",  "partitioned" : true,  "version" : {  "uuid" : UUID("c85b8bea-bcb8-409b-b969-8c7f4ed8a1c4"),  "lastMod" : 1 } }
                test1.users
                        shard key: { "name" : "hashed" }   # test1.users.name进行hashed路由
                        unique: false
                        balancing: true
                        chunks:
                                shard-rs02      2
                                shard-rs03      2
                        { "name" : { "$minKey" : 1 } } -->> { "name" : NumberLong("-4611686018427387902") } on : shard-rs02 Timestamp(1, 0)
                        { "name" : NumberLong("-4611686018427387902") } -->> { "name" : NumberLong(0) } on : shard-rs02 Timestamp(1, 1)
                        { "name" : NumberLong(0) } -->> { "name" : NumberLong("4611686018427387902") } on : shard-rs03 Timestamp(1, 2)
                        { "name" : NumberLong("4611686018427387902") } -->> { "name" : { "$maxKey" : 1 } } on : shard-rs03 Timestamp(1, 3)
```
### (20). 分片测试
```
# 1. hash分片(总共插入1000条数据)
mongos> use test1;
mongos> for(var i=0;i<1000;i++){ if(i%2==0) { db.users.insert({name:"张三" + i,age:20,gender:"男"}); } else { db.users.insert({name:"张三"+i ,age:20,gender:"女"}); } }

# 2. 验证hash分片结果(两个副本集的数量之和,要求达到:1000条)
# shard-rs02 副本集数据
shard-rs02:PRIMARY> show dbs;
test1   0.000GB
shard-rs02:PRIMARY> use test1;
switched to db test1
shard-rs02:PRIMARY> db.users.count();
507   

# shard-rs03 副本集数据
shard-rs03:PRIMARY> show dbs;
test1   0.000GB
shard-rs03:PRIMARY> use test1;
switched to db test1
shard-rs03:PRIMARY> db.users.count();
493
```
### (21). 

### (22). 

### (23). 

### (24). 

### (25). 