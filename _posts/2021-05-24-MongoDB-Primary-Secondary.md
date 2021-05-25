---
layout: post
title: 'MongoDB 副本集搭建(二)'
date: 2021-05-24
author: 李新
tags:  MongoDB
---

### (1). 机器准备

|  机器名称          | IP            |   角色      |
|  ----            | ----           |  ----      |
|  clickhouse-1    | 10.211.55.100  |  Primary   |
|  clickhouse-2    | 10.211.55.101  |  Secondary |
|  clickhouse-3    | 10.211.55.102  |  Arbiter   |

### (2). 下载mongodb
```
# 1. 当前工作目录
[root@clickhouse-N soft]# pwd
/opt/soft
# 2. 下载mongodb
[root@clickhouse-N soft]# wget https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-rhel70-4.4.6.tgz

# 3. 解压
[root@clickhouse-N soft]# tar -zxvf mongodb-linux-x86_64-rhel70-4.4.6.tgz

# 4. 创建数据目录/日志目录/配置文件夹目录
[root@clickhouse-N soft]# mkdir -p mongodb-linux-x86_64-rhel70-4.4.6/data
[root@clickhouse-N soft]# mkdir -p mongodb-linux-x86_64-rhel70-4.4.6/logs
[root@clickhouse-N soft]# mkdir -p mongodb-linux-x86_64-rhel70-4.4.6/conf
```
### (3). mongod.conf
```
systemLog:
    destination: file
    path: "/opt/soft/mongodb-linux-x86_64-rhel70-4.4.6/logs/mongodb.log"
    logAppend: true
storage:
    dbPath: "/opt/soft/mongodb-linux-x86_64-rhel70-4.4.6/data"
    journal:  # 启用持久化日志以确保数据文件保持有效和可恢复.
        enabled: true
processManagement:
    fork: true
    pidFilePath: "/opt/soft/mongodb-linux-x86_64-rhel70-4.4.6/logs/mongodb.pid"
net:
    bindIp: 0.0.0.0
    port: 27017

# 副本集的名称(三台机器要保持一持)
replication:
    replSetName: cluster    
```
### (4). 所有的节点都启动
```
# clickhouse-1
[root@clickhouse-1 bin]# ./mongod -f ../conf/mongod.conf
about to fork child process, waiting until server is ready for connections.
forked process: 9292
child process started successfully, parent exiting

# clickhouse-2
[root@clickhouse-2 bin]# ./mongod  -f ../conf/mongod.conf
about to fork child process, waiting until server is ready for connections.
forked process: 9697
child process started successfully, parent exiting

# clickhouse-3
[root@clickhouse-3 bin]# ./mongod  -f ../conf/mongod.conf
about to fork child process, waiting until server is ready for connections.
forked process: 9752
child process started successfully, parent exiting
```
### (5). 初始化副本集
```
# 注意,该操作是在:clickhouse-1节点执行.
[root@clickhouse-1 bin]# ./mongo --host=127.0.0.1 --port=27017
MongoDB shell version v4.4.6

# 1.初始化副本集
> rs.initiate();
{
        "info2" : "no configuration specified. Using a default configuration for the set",
        "me" : "clickhouse-1:27017",
        "ok" : 1   # 初台化成功
}
cluster:SECONDARY>

# 2. 查看默认的配置信息
cluster:PRIMARY> rs.conf()
{
        "_id" : "cluster",
        "version" : 1,
        "term" : 1,
        "protocolVersion" : NumberLong(1),
        "writeConcernMajorityJournalDefault" : true,
		# 集群成员
        "members" : [
                {
                        "_id" : 0,
                        "host" : "clickhouse-1:27017",
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
                "replicaSetId" : ObjectId("60aca30bb77331d2485ed4f9")
        }
}

# 3. 查看当前节点运行状态
cluster:PRIMARY> rs.status();
{
        "set" : "cluster",
        "date" : ISODate("2021-05-25T07:15:17.319Z"),
        "myState" : 1,
        "term" : NumberLong(1),
        "syncSourceHost" : "",
        "syncSourceId" : -1,
        "heartbeatIntervalMillis" : NumberLong(2000),
        "majorityVoteCount" : 1,
        "writeMajorityCount" : 1,
        "votingMembersCount" : 1,
        "writableVotingMembersCount" : 1,
        "optimes" : {
                "lastCommittedOpTime" : {
                        "ts" : Timestamp(1621926907, 1),
                        "t" : NumberLong(1)
                },
                "lastCommittedWallTime" : ISODate("2021-05-25T07:15:07.661Z"),
                "readConcernMajorityOpTime" : {
                        "ts" : Timestamp(1621926907, 1),
                        "t" : NumberLong(1)
                },
                "readConcernMajorityWallTime" : ISODate("2021-05-25T07:15:07.661Z"),
                "appliedOpTime" : {
                        "ts" : Timestamp(1621926907, 1),
                        "t" : NumberLong(1)
                },
                "durableOpTime" : {
                        "ts" : Timestamp(1621926907, 1),
                        "t" : NumberLong(1)
                },
                "lastAppliedWallTime" : ISODate("2021-05-25T07:15:07.661Z"),
                "lastDurableWallTime" : ISODate("2021-05-25T07:15:07.661Z")
        },
        "lastStableRecoveryTimestamp" : Timestamp(1621926897, 1),
        "electionCandidateMetrics" : {
                "lastElectionReason" : "electionTimeout",
                "lastElectionDate" : ISODate("2021-05-25T07:11:07.595Z"),
                "electionTerm" : NumberLong(1),
                "lastCommittedOpTimeAtElection" : {
                        "ts" : Timestamp(0, 0),
                        "t" : NumberLong(-1)
                },
                "lastSeenOpTimeAtElection" : {
                        "ts" : Timestamp(1621926667, 1),
                        "t" : NumberLong(-1)
                },
                "numVotesNeeded" : 1,
                "priorityAtElection" : 1,
                "electionTimeoutMillis" : NumberLong(10000),
                "newTermStartDate" : ISODate("2021-05-25T07:11:07.613Z"),
                "wMajorityWriteAvailabilityDate" : ISODate("2021-05-25T07:11:07.645Z")
        },
        "members" : [
                {
                        "_id" : 0,
                        "name" : "clickhouse-1:27017",
                        "health" : 1,
                        "state" : 1,
                        "stateStr" : "PRIMARY",
                        "uptime" : 726,
                        "optime" : {
                                "ts" : Timestamp(1621926907, 1),
                                "t" : NumberLong(1)
                        },
                        "optimeDate" : ISODate("2021-05-25T07:15:07Z"),
                        "syncSourceHost" : "",
                        "syncSourceId" : -1,
                        "infoMessage" : "",
                        "electionTime" : Timestamp(1621926667, 2),
                        "electionDate" : ISODate("2021-05-25T07:11:07Z"),
                        "configVersion" : 1,
                        "configTerm" : 1,
                        "self" : true,
                        "lastHeartbeatMessage" : ""
                }
        ],
        "ok" : 1,
        "$clusterTime" : {
                "clusterTime" : Timestamp(1621926907, 1),
                "signature" : {
                        "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
                        "keyId" : NumberLong(0)
                }
        },
        "operationTime" : Timestamp(1621926907, 1)
}

# 4. 添加副本从节点(Secondary)   
# rs.add()添加副本从节点
cluster:PRIMARY> rs.add("10.211.55.101:27017");
{
        "ok" : 1,                 # 添加副本从节点成功
        "$clusterTime" : {
                "clusterTime" : Timestamp(1621927271, 1),
                "signature" : {
                        "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
                        "keyId" : NumberLong(0)
                }
        },
        "operationTime" : Timestamp(1621927271, 1)
}


# 5. 添加仲裁节点(Arbiter)
cluster:PRIMARY> rs.addArb("10.211.55.102:27017");
{
        "ok" : 1,
        "$clusterTime" : {
                "clusterTime" : Timestamp(1621927417, 1),
                "signature" : {
                        "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
                        "keyId" : NumberLong(0)
                }
        },
        "operationTime" : Timestamp(1621927417, 1)
}

# 6. 查看副本集配置
cluster:PRIMARY> rs.conf();
{
        "_id" : "cluster",
        "version" : 3,
        "term" : 1,
        "protocolVersion" : NumberLong(1),
        "writeConcernMajorityJournalDefault" : true,
        "members" : [
                {
                        "_id" : 0,
                        "host" : "clickhouse-1:27017",
                        "arbiterOnly" : false,
                        "buildIndexes" : true,
                        "hidden" : false,
                        "priority" : 1,                    # 投票数量为1票
                        "tags" : {

                        },
                        "slaveDelay" : NumberLong(0),
                        "votes" : 1
                },
                {
                        "_id" : 1,
                        "host" : "10.211.55.101:27017",
                        "arbiterOnly" : false,
                        "buildIndexes" : true,
                        "hidden" : false,
                        "priority" : 1,                   # 投票数量为1票
                        "tags" : {

                        },
                        "slaveDelay" : NumberLong(0),
                        "votes" : 1
                },
                {
                        "_id" : 2,
                        "host" : "10.211.55.102:27017",
                        "arbiterOnly" : true,
                        "buildIndexes" : true,
                        "hidden" : false,
                        "priority" : 0,                       # 10.211.55.102:27017节点不参与投票
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
                "replicaSetId" : ObjectId("60aca30bb77331d2485ed4f9")
        }
}
```
### (6). 副本集的数据读写
```

cluster:PRIMARY> use test;
switched to db test

# 1. 在主节点上的:test库下创建集合:users
cluster:PRIMARY> db.users.insert([{name:"张三",age:20,gender:"男"},{name:"李四",age:21,gender:"女"}]);
BulkWriteResult({
        "writeErrors" : [ ],
        "writeConcernErrors" : [ ],
        "nInserted" : 2,
        "nUpserted" : 0,
        "nMatched" : 0,
        "nModified" : 0,
        "nRemoved" : 0,
        "upserted" : [ ]
})

# 2. 在主节点,查看数据状态
cluster:PRIMARY> db.users.find();
{ "_id" : ObjectId("60aca74473e0586b635a155e"), "name" : "张三", "age" : 20, "gender" : "男" }
{ "_id" : ObjectId("60aca74473e0586b635a155f"), "name" : "李四", "age" : 21, "gender" : "女" }


# 3. 登录到从节点
[root@clickhouse-2 bin]# ./mongo --host=127.0.0.1 --port=27017
MongoDB shell version v4.4.6

# 4. 查看数据库
cluster:SECONDARY> show dbs;
uncaught exception: Error: listDatabases failed:{
        "topologyVersion" : {
                "processId" : ObjectId("60aca1cd77ad22500e3ca825"),
                "counter" : NumberLong(4)
        },
        "operationTime" : Timestamp(1621927927, 1),
        "ok" : 0,
        "errmsg" : "not master and slaveOk=false",   # 
        "code" : 13435,
        "codeName" : "NotPrimaryNoSecondaryOk",    
        "$clusterTime" : {
                "clusterTime" : Timestamp(1621927927, 1),
                "signature" : {
                        "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
                        "keyId" : NumberLong(0)
                }
        }
} :
_getErrorWithCode@src/mongo/shell/utils.js:25:13
Mongo.prototype.getDBs/<@src/mongo/shell/mongo.js:147:19
Mongo.prototype.getDBs@src/mongo/shell/mongo.js:99:12
shellHelper.show@src/mongo/shell/utils.js:937:13
shellHelper@src/mongo/shell/utils.js:819:15
@(shellhelp2):1:1

####################################################################################
# 5. 设置自己为slave节点
#  rs.slaveOk(true)  true->设置为slave   false->取消slave
####################################################################################
cluster:SECONDARY> rs.slaveOk(true);
WARNING: slaveOk() is deprecated and may be removed in the next major release. Please use secondaryOk() instead.

# 6. 重新执行查看数据库
cluster:SECONDARY> show dbs;
admin   0.000GB
config  0.000GB
local   0.000GB
test    0.000GB

# 7. 进入test库
cluster:SECONDARY> use test;
switched to db test

# 8. 查看所有的集合
cluster:SECONDARY> show collections;
users

# 9. 查询集合列表
cluster:SECONDARY> db.users.find();
{ "_id" : ObjectId("60aca74473e0586b635a155e"), "name" : "张三", "age" : 20, "gender" : "男" }
{ "_id" : ObjectId("60aca74473e0586b635a155f"), "name" : "李四", "age" : 21, "gender" : "女" }
```
### (7). 查看仲裁节点
```
[root@clickhouse-3 bin]# ./mongo --host=127.0.0.1 --port=27017
MongoDB shell version v4.4.6
# 1. 允许成为slave
cluster:ARBITER> rs.slaveOk();
WARNING: slaveOk() is deprecated and may be removed in the next major release. Please use secondaryOk() instead.

# 2. 查看有多少库
cluster:ARBITER> show dbs
uncaught exception: Error: listDatabases failed:{
        "topologyVersion" : {
                "processId" : ObjectId("60aca1e1dcc7d8a1c7163582"),
                "counter" : NumberLong(1)
        },
        "ok" : 0,
        "errmsg" : "node is not in primary or recovering state",   # 不允许查看,节点非primary而且还不接受数据状态.
        "code" : 13436,
        "codeName" : "NotPrimaryOrSecondary"
} :
_getErrorWithCode@src/mongo/shell/utils.js:25:13
Mongo.prototype.getDBs/<@src/mongo/shell/mongo.js:147:19
Mongo.prototype.getDBs@src/mongo/shell/mongo.js:99:12
shellHelper.show@src/mongo/shell/utils.js:937:13
shellHelper@src/mongo/shell/utils.js:819:15
@(shellhelp2):1:1
```
### (8). 总结
> 主从复制选举规则为:  
> 1. 必须半数以上的仲裁节点投票通过.  
> 2. 新的主节点为最新数据的节点.   