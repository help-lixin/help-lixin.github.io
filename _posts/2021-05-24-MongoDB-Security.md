---
layout: post
title: 'MongoDB 安全管理(五)'
date: 2021-05-24
author: 李新
tags:  MongoDB
---

### (1). 概述
> 在这里,主要对MongoDB的安全进行配置.

### (2). MongoDB DCL

```
> show dbs;
admin   0.000GB
config  0.000GB
local   0.000GB

# 1. 切换到admin
> use admin;
switched to db admin

# 2. 创建普通管理员(admin)
> db.createUser({user:"super-admin",pwd:"123456",roles:[ { role:"userAdminAnyDatabase" , db:"admin"} ]});
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
> use test;
> db.createUser({user:"lixin",pwd:"123456",roles:[ { role:"readWrite" , db:"test"} ]});
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
> use admin;
> db.system.users.find();
{ "_id" : "admin.super-admin", "userId" : UUID("aa593123-ba68-475c-8f3c-9beb0fe20451"), "user" : "super-admin", "db" : "admin", "credentials" : { "SCRAM-SHA-1" : { "iterationCount" : 10000, "salt" : "KiY+BoTjbIHXy5GpmUJ5Gg==", "storedKey" : "U0PiAfNAjPBXE/QkkOGeB+TeWoY=", "serverKey" : "6Wg2w/KEU8Fuc7naon1F4GbdlC8=" }, "SCRAM-SHA-256" : { "iterationCount" : 15000, "salt" : "qjR8Wa5miU7pYLzxzleB9MIR0bRKR78vxphKVA==", "storedKey" : "yIobZBOpvPtPceUX3z/YCPo8r3rscCrm67j4QW2bwz0=", "serverKey" : "2iCKX8JvCCUzw1IjfFdvH+7CZIvQ14U7WxCl1pTCAc4=" } }, "roles" : [ { "role" : "userAdminAnyDatabase", "db" : "admin" } ] }
{ "_id" : "test.lixin", "userId" : UUID("ae650d26-6b27-4067-b2b0-afd9501a76cb"), "user" : "lixin", "db" : "test", "credentials" : { "SCRAM-SHA-1" : { "iterationCount" : 10000, "salt" : "4b+MUJTn9NbKtOC2D3M8bw==", "storedKey" : "LCTFfcO8VwGQJgJfyjnK8on336A=", "serverKey" : "/YsvY9hkIjo9kWnSuNtM19xf/xg=" }, "SCRAM-SHA-256" : { "iterationCount" : 15000, "salt" : "u13qTI3DgmnzhVt8F1bmWK+FSeJPrO4iJgWMzw==", "storedKey" : "tfkcAYW/OWFpmxo8QnG7JAdNoGLFayFRFNsahM1ZzGY=", "serverKey" : "dOzpWz7/141K3wav9jjac8QZHiN/cUq3UtPIBSBNJUY=" } }, "roles" : [ { "role" : "readWrite", "db" : "test" } ] }

# 5. 修改用户密码
> db.changeUserPassword("super-admin","111111");

# 6. 删除刚创建的用户(admin)
> db.dropUser('super-admin');

# 7. 测试账号和密码
> db.auth('root','111111')
1

> db.auth('root','123456')
Error: Authentication failed.
0
```
### (3). MongoDB单机模式下,配置启用安全认证(mongod.conf)
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

security:
    authorization: enabled

net:
    bindIp: 0.0.0.0
    port: 27017
```
### (4). 测试
```
# lixin-macbook:mongodb-macos-x86_64-4.4.1 lixin$ ./bin/mongo test -u lixin -p 123456
lixin-macbook:mongodb-macos-x86_64-4.4.1 lixin$ ./bin/mongo
MongoDB shell version v4.4.1
connecting to: mongodb://127.0.0.1:27017/?compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("17631778-0b71-4ea5-a4d0-9067233f43e1") }
MongoDB server version: 4.4.1

> show dbs;

# 1. 登录
> db.auth('lixin','123456');
1

# 2. 查看当前所在的库
> db
test

# 3. 插入三条数据
> db.users.insert({name:"张三",age:20,gender:"男"});
WriteResult({ "nInserted" : 1 })

> db.users.insert([{name:"李四",age:21,gender:"男"},{name:"王五",age:22,gender:"女"}]);
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

# 4. 查看有多少Collection
> show collections;
users

# 5. Query
> db.users.find();
{ "_id" : "hello", "name" : "hello", "age" : 18, "gender" : "男" }
{ "_id" : ObjectId("60af3d0aaff5e64865b6f4ed"), "name" : "李四", "age" : 21, "gender" : "男" }
{ "_id" : ObjectId("60af3d0aaff5e64865b6f4ee"), "name" : "王五", "age" : 22, "gender" : "女" }


# 6. 在切换到管理员之前,必须先进入到:admin库,否则:会提示:Error: Authentication failed.
> use admin;
switched to db admin
> db.auth("root","111111");
1
```

### (5). spring-data-mongodb配置账号和密码
```
# spring.data.mongodb.username=lixin
# spring.data.mongodb.password=123456
# 不能再像上面那样配置账号和密码了,需要像下面这样配置.
spring.data.mongodb.uri=mongodb://lixin:123456@127.0.0.1:27017/test
```
### (5). 总结
> MongoDB的授权过程还是挺麻烦的,在对某个库授权时,必须要先进入该库,否则,在登录时,出现:Authentication failed,这个折腾了我很久.      