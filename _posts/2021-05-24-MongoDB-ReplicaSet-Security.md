---
layout: post
title: 'MongoDB 副本集安全管理(六)'
date: 2021-05-24
author: 李新
tags:  MongoDB
---

### (1). 概述
> 在这里,主要对MongoDB的副本集安全进行配置.

### (2). MongoDB DCL操作
> 在Primary节点创建普通账号.  

```
lixin-macbook:mongodb-replication lixin$ ./mongodb-macos-x86_64-4.4.1/bin/mongo --port 27017

repl-rs:PRIMARY> show dbs;
admin   0.000GB
config  0.000GB
local   0.000GB

# 1. 切换到admin
repl-rs:PRIMARY> use admin;
switched to db admin

# 2. 创建普通管理员(admin)
repl-rs:PRIMARY> db.createUser({user:"super-admin",pwd:"123456",roles:[ { role:"userAdminAnyDatabase" , db:"admin"} ]});
Successfully added user: {
	"user" : "super-admin",
	"roles" : [
		{
			"role" : "userAdminAnyDatabase",
			"db" : "admin"
		}
	]
}

# ******************************************************************************
# 3.创建普通用户(test,只能对test库进行读写操作),要对哪个库授权,记得一定要切到对应的库.
# ******************************************************************************
repl-rs:PRIMARY> use test;
repl-rs:PRIMARY> db.createUser({user:"lixin",pwd:"123456",roles:[ { role:"readWrite" , db:"test"} ]});
Successfully added user: {
	"user" : "lixin",
	"roles" : [
		{
			"role" : "readWrite",
			"db" : "test"
		}
	]
}

# 4. 查看创建了多少用户
repl-rs:PRIMARY> use admin;
{ "_id" : "admin.super-admin", "userId" : UUID("9f6cd76f-cdc8-4872-8915-45f377f44ea1"), "user" : "super-admin", "db" : "admin", "credentials" : { "SCRAM-SHA-1" : { "iterationCount" : 10000, "salt" : "A5QliKZ+NH3ssKPGSp7+VA==", "storedKey" : "G2lMgH+AdESLtMeFWadSMuBCCTU=", "serverKey" : "l+/HPpPPLMGgSSmOPlLk7pS4UFo=" }, "SCRAM-SHA-256" : { "iterationCount" : 15000, "salt" : "nKygQjnmuo/kxyRtRgbwvbtQI6Wpi+1DAj6fJw==", "storedKey" : "91UnZhyryXRtCK0Vp9wZs+07s700ow5Pvy/x/1BtElc=", "serverKey" : "/Ev78IN/XdR8NAjgzkfiY0jGG7NsU2DeHlKSyfiIuhY=" } }, "roles" : [ { "role" : "userAdminAnyDatabase", "db" : "admin" } ] }
{ "_id" : "test.lixin", "userId" : UUID("75ba04bd-a38f-496e-b554-e07b48050e31"), "user" : "lixin", "db" : "test", "credentials" : { "SCRAM-SHA-1" : { "iterationCount" : 10000, "salt" : "esNqF3kS8G6Ee+oYbM6D3w==", "storedKey" : "z90mDFvyhvs2WSwLQJ5Z9Q/Ixww=", "serverKey" : "Y3OVp5cZGQQEHZCn1z4z51BoCaU=" }, "SCRAM-SHA-256" : { "iterationCount" : 15000, "salt" : "VUT0o8Ynr7EBENnRGN/nJ4obNs3uWGfUCBJQOw==", "storedKey" : "vFEQBhDX8Xswup7sMhRLOxnSf308bpsVZQp0yvru6GU=", "serverKey" : "9VWjB+VmGVa6pW1dHbxnLgdMuFO4GxMlUWJpB85M8bk=" } }, "roles" : [ { "role" : "readWrite", "db" : "test" } ] }

# 5. 修改用户密码
repl-rs:PRIMARY> db.changeUserPassword("super-admin","123456");

# 6. 删除刚创建的用户(admin)
# repl-rs:PRIMARY> db.dropUser('super-admin');

# 7. 测试账号和密码
# repl-rs:PRIMARY> db.auth('root','111111')
# 1

repl-rs:PRIMARY> db.auth('super-admin','1234567')
Error: Authentication failed.
0
```
### (3). 创建副本集认证的key文件
```
# ****注意:整个副本集要共用同一个key文件.****
# 1. 查看当前所在的目录
lixin-macbook:mongodb-replication lixin$ pwd
/Users/lixin/Developer/mongodb-cluster/mongodb-replication
# 2. 通过openssl生成key文件
lixin-macbook:mongodb-replication lixin$ openssl rand  -base64 90 -out  ./mongo.keyfile
# 3. 必须修改key文件为可读
lixin-macbook:mongodb-replication lixin$ chmod 400 ./mongo.keyfile
```
### (4). MongoDB副本集模式下,配置启用安全认证和key文件(mongod.conf)
```
systemLog:
    destination: file
    path: "/Users/lixin/Developer/mongodb-cluster/mongodb-macos-x86_64-4.4.1/logs/mongodb.log"
    logAppend: true
storage:
    dbPath: "/Users/lixin/Developer/mongodb-cluster/mongodb-macos-x86_64-4.4.1/data"
    journal:  # 启用持久化日志以确保数据文件保持有效和可恢复.
        enabled: true
processManagement:
    fork: true
    pidFilePath: "/Users/lixin/Developer/mongodb-cluster/mongodb-macos-x86_64-4.4.1/logs/mongodb.pid"

# 添加keyfile,以及打开安全管理
security:
    keyFile: /Users/lixin/Developer/mongodb-cluster/mongodb-replication/mongo.keyfile
    authorization: enabled

net:
    bindIp: 0.0.0.0
    port: 27017
```
### (4). 测试
```
# 连接到主节点
lixin-macbook:mongodb-replication lixin$ ./mongodb-macos-x86_64-4.4.1/bin/mongo --port 27018
MongoDB shell version v4.4.1
connecting to: mongodb://127.0.0.1:27018/?compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("543e28a3-35cc-42a2-a75f-f3ba26629fe6") }
MongoDB server version: 4.4.1
repl-rs:PRIMARY> show dbs;

# 1. 登录
repl-rs:PRIMARY> db.auth("lixin","123456");
1

# 2. 查看当前所在的数据库
repl-rs:PRIMARY> db
test

# 3. 查看所有的集合
repl-rs:PRIMARY> show collections
users

# 4. Query
repl-rs:PRIMARY> db.users.find();
{ "_id" : ObjectId("60af98dcce11d8176ea73e77"), "name" : "张三", "age" : 20, "gender" : "男" }
{ "_id" : ObjectId("60af98e0ce11d8176ea73e79"), "name" : "王五", "age" : 22, "gender" : "女" }
{ "_id" : ObjectId("60af98e0ce11d8176ea73e78"), "name" : "李四", "age" : 21, "gender" : "男" }
```
### (5). spring-data-mongodb配置账号和密码
```
# spring.data.mongodb.username=lixin
# spring.data.mongodb.password=123456
# replicaSet=repl-rs  : 用于指定副本集的唯一名称
spring.data.mongodb.uri=mongodb://lixin:123456@127.0.0.1:27017,127.0.0.1:27018,127.0.0.1:27019/test?connect=replicaSet&slaveOk=true&replicaSet=repl-rs
```
### (5). 总结
