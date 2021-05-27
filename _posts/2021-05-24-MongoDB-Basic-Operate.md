---
layout: post
title: 'MongoDB 基础入门(一)'
date: 2021-05-24
author: 李新
tags:  MongoDB
---

### 1. 单机安装和启动MongoDB
```
# 1. 查看当前目录
lixin-macbook:bin lixin$ pwd
/Users/lixin/Developer/mongodb-cluster/mongodb-macos-x86_64-4.4.1/bin


# 2. 启动Mongod,并指定数据目录以及port
lixin-macbook:bin lixin$ ./mongod --dbpath /Users/lixin/Developer/mongodb-cluster/mongodb-macos-x86_64-4.4.1/data  --port 27017

# 3. 设计启动脚本.
# #!/bin/bash
# nohup /Users/lixin/Developer/mongodb-cluster/mongodb-macos-x86_64-4.4.1/bin/mongod --dbpath /Users/lixin/Developer/mongodb-cluster/mongodb-macos-x86_64-4.4.1/data > /dev/null 2>&1 &
```
### 2.  启动MongoDB client
```
# 1. 查看当前目录
lixin-macbook:bin lixin$ pwd
/Users/lixin/Developer/mongodb-cluster/mongodb-macos-x86_64-4.4.1/bin

# 2. 启动客户端.
lixin-macbook:bin lixin$ ./mongo
MongoDB shell version v4.4.1
connecting to: mongodb://127.0.0.1:27017/?compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("5721a5da-f0cc-4894-9c3f-252f07588ceb") }
MongoDB server version: 4.4.1
>
```
### 3. MongoDB常用操作
```
# 1.显示当前所有数据库
> show databases;
admin   0.000GB
config  0.000GB
local   0.000GB

# 2.进入到(test)数据库里(MongoDB中数据库和集合都不需要手工先创建)
> use test;
switched to db test

# 3.查看当前所在的数据库(有点类似于:db==this)
> db;
test

# 4. 查看test库下,有哪些集合
> show collections;


# 5. 删除数据库(当前所在的数据库)
> db.dropDatabase();


# 6. 删除collection
> db.users.drop();
```
### 4. MongoDB 插入文档
```
# 1.插入一个文档
> db.users.insert({_id:"hello",name:"hello",age:18,gender:"男"});
WriteResult({ "nInserted" : 1 })

> db.users.insert({name:"张三",age:20,gender:"男"});
WriteResult({ "nInserted" : 1 })

# 2. 插入多个文档
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

```
### 5. MongoDB 查询文档
```
# 1. 查询users集合下所有的文档.
> db.users.find();
{ "_id" : "hello", "name" : "hello", "age" : 18, "gender" : "男" }
{ "_id" : ObjectId("60ab58c68453d1ab4a650c01"), "name" : "张三", "age" : 20, "gender" : "男" }
{ "_id" : ObjectId("60ab58cb8453d1ab4a650c02"), "name" : "李四", "age" : 21, "gender" : "男" }
{ "_id" : ObjectId("60ab58cb8453d1ab4a650c03"), "name" : "王五", "age" : 22, "gender" : "女" }

# 2. 根据_id查询
> db.users.find({_id:"hello"});
{ "_id" : "hello", "name" : "hello", "age" : 18, "gender" : "男" }

# 3. WHERE name = "张三" and age = 20
> db.users.find({name:"张三",age:20});
{ "_id" : ObjectId("60ab58c68453d1ab4a650c01"), "name" : "张三", "age" : 20, "gender" : "男" }


# $lt   :  < 
# $lte  :  <= 
# $gt   :  > 
# $gte  :  >= 
# $ne   :  !=  

# 4. WHERE age < 21
> db.users.find({age:{ $lt:21} });
{ "_id" : "hello", "name" : "hello", "age" : 18, "gender" : "男" }
{ "_id" : ObjectId("60ab58c68453d1ab4a650c01"), "name" : "张三", "age" : 20, "gender" : "男" }

# 5. WHERE age > 21
> db.users.find({age:{$gt:21}});
{ "_id" : ObjectId("60ab58cb8453d1ab4a650c03"), "name" : "王五", "age" : 22, "gender" : "女" }

# 6. WHERE (name = "张三") OR (age < 21)
> db.users.find({ $or:[ { name:"张三" } , { age :{ $lt : 21 } } ] });
{ "_id" : "hello", "name" : "hello", "age" : 18, "gender" : "男" }
{ "_id" : ObjectId("60ab58c68453d1ab4a650c01"), "name" : "张三", "age" : 20, "gender" : "男" }

# 7. WHERE gender = "男" AND ( age > 18 OR age < 22 )
> db.users.find({ gender:"男" , $or:[ {age:{$gt:18 , $lt: 22}} ]  });
{ "_id" : ObjectId("60ab58c68453d1ab4a650c01"), "name" : "张三", "age" : 20, "gender" : "男" }
{ "_id" : ObjectId("60ab58cb8453d1ab4a650c02"), "name" : "李四", "age" : 21, "gender" : "男" }


# 8. WHERE gender = "男" AND ( age > 18 OR age < 22 ) LIMIT 1
> db.users.findOne({ gender:"男" , $or:[ {age:{$gt:18 , $lt: 22}} ]  });
{
	"_id" : ObjectId("60ab58c68453d1ab4a650c01"),
	"name" : "张三",
	"age" : 20,
	"gender" : "男"
}

# 9. 统计有多少文档
> db.users.find().count();
4

# 10. 分页查询前,先查看所有的数据.
> db.users.find();
{ "_id" : ObjectId("60ab58c68453d1ab4a650c01"), "name" : "张三", "age" : 19, "gender" : "男" }
{ "_id" : ObjectId("60ab58cb8453d1ab4a650c02"), "name" : "李四", "age" : 21, "gender" : "男" }
{ "_id" : ObjectId("60ab58cb8453d1ab4a650c03"), "name" : "王五", "age" : 22, "gender" : "女" }
{ "_id" : ObjectId("60ab6b5e352018b15d71a152"), "name" : "赵六", "age" : 20, "gender" : "女" }

# 分页查询(每页显示两条)
> db.users.find().skip(0).limit(2);
{ "_id" : ObjectId("60ab58c68453d1ab4a650c01"), "name" : "张三", "age" : 19, "gender" : "男" }
{ "_id" : ObjectId("60ab58cb8453d1ab4a650c02"), "name" : "李四", "age" : 21, "gender" : "男" }

# 分页查询(每页显示两条)
> db.users.find().skip(2).limit(2);
{ "_id" : ObjectId("60ab58cb8453d1ab4a650c03"), "name" : "王五", "age" : 22, "gender" : "女" }
{ "_id" : ObjectId("60ab6b5e352018b15d71a152"), "name" : "赵六", "age" : 20, "gender" : "女" }


# 11. 按年龄的降序排序
> db.users.find();
{ "_id" : ObjectId("60ab58c68453d1ab4a650c01"), "name" : "张三", "age" : 19, "gender" : "男" }
{ "_id" : ObjectId("60ab58cb8453d1ab4a650c02"), "name" : "李四", "age" : 21, "gender" : "男" }
{ "_id" : ObjectId("60ab58cb8453d1ab4a650c03"), "name" : "王五", "age" : 22, "gender" : "女" }
{ "_id" : ObjectId("60ab6b5e352018b15d71a152"), "name" : "赵六", "age" : 20, "gender" : "女" }


# 年龄降序排序(1:升序 -1:降序)
> db.users.find().sort({age:-1})
{ "_id" : ObjectId("60ab58cb8453d1ab4a650c03"), "name" : "王五", "age" : 22, "gender" : "女" }
{ "_id" : ObjectId("60ab58cb8453d1ab4a650c02"), "name" : "李四", "age" : 21, "gender" : "男" }
{ "_id" : ObjectId("60ab6b5e352018b15d71a152"), "name" : "赵六", "age" : 20, "gender" : "女" }
{ "_id" : ObjectId("60ab58c68453d1ab4a650c01"), "name" : "张三", "age" : 19, "gender" : "男" }


# 按照age降序排序,并分页
> db.users.find().sort({age:-1}).skip(0).limit(2);
{ "_id" : ObjectId("60ab58cb8453d1ab4a650c03"), "name" : "王五", "age" : 22, "gender" : "女" }
{ "_id" : ObjectId("60ab58cb8453d1ab4a650c02"), "name" : "李四", "age" : 21, "gender" : "男" }


# 按照age降序排序,并分页
> db.users.find().sort({age:-1}).skip(2).limit(2);
{ "_id" : ObjectId("60ab6b5e352018b15d71a152"), "name" : "赵六", "age" : 20, "gender" : "女" }
{ "_id" : ObjectId("60ab58c68453d1ab4a650c01"), "name" : "张三", "age" : 19, "gender" : "男" }

# 12. 查询结果,仅显示:name字段.
> db.users.find({},{name:1,_id:0});
{ "_id" : ObjectId("60ab58c68453d1ab4a650c01"), "name" : "张三" }
{ "_id" : ObjectId("60ab58cb8453d1ab4a650c02"), "name" : "李四" }
{ "_id" : ObjectId("60ab58cb8453d1ab4a650c03"), "name" : "王五" }
{ "_id" : ObjectId("60ab6b5e352018b15d71a152"), "name" : "赵六" }


# 13. 模糊查询
# 查询name包含有:"三"的数据
> db.users.find({name:/三/});
{ "_id" : ObjectId("60ae46404ba99318ae55742a"), "name" : "张三", "age" : 20, "gender" : "男" }


# 查询name包含以:"张"开头的数据
> db.users.find({name:/^张/});
{ "_id" : ObjectId("60ae46404ba99318ae55742a"), "name" : "张三", "age" : 20, "gender" : "男" }
```
### 6. MongoDB 修改文档
> 注意:update默认情况下,会使用"新的对象"来替换我"旧的对象".   
> 如果需要的是修改文档,则需要使用"修改操作符".   
> update默认情况下,只会修入一条数据(可通过参数配置),而,updateMany则可以修改多条数据.  

> https://docs.mongodb.com/manual/reference/method/db.collection.update/#mongodb-method-db.collection.update   


```
# 1.根据id修改文档局部内容
> db.users.update( {"_id" : ObjectId("60ab58c68453d1ab4a650c01")} , { $set:{ age:19 } } );
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })

# 2. 查询出所有数据,验证
> db.users.find();
{ "_id" : "hello", "name" : "hello", "age" : 18, "gender" : "男" }
{ "_id" : ObjectId("60ab58c68453d1ab4a650c01"), "name" : "张三", "age" : 19, "gender" : "男" }
{ "_id" : ObjectId("60ab58cb8453d1ab4a650c02"), "name" : "李四", "age" : 21, "gender" : "男" }
{ "_id" : ObjectId("60ab58cb8453d1ab4a650c03"), "name" : "王五", "age" : 22, "gender" : "女" }
```

### 7. MongoDB 删除文档
> https://docs.mongodb.com/manual/reference/method/db.collection.remove/  

```
# 1. 删除之前查看数据
> db.users.find();
{ "_id" : "hello", "name" : "hello", "age" : 18, "gender" : "男" }
{ "_id" : ObjectId("60ab58c68453d1ab4a650c01"), "name" : "张三", "age" : 19, "gender" : "男" }
{ "_id" : ObjectId("60ab58cb8453d1ab4a650c02"), "name" : "李四", "age" : 21, "gender" : "男" }
{ "_id" : ObjectId("60ab58cb8453d1ab4a650c03"), "name" : "王五", "age" : 22, "gender" : "女" }

# 2. 根据文档ID,删除文档
> db.users.remove({"_id" : "hello"});
WriteResult({ "nRemoved" : 1 })

# 3. 删除之后查看数据
> db.users.find();
{ "_id" : ObjectId("60ab58c68453d1ab4a650c01"), "name" : "张三", "age" : 19, "gender" : "男" }
{ "_id" : ObjectId("60ab58cb8453d1ab4a650c02"), "name" : "李四", "age" : 21, "gender" : "男" }
{ "_id" : ObjectId("60ab58cb8453d1ab4a650c03"), "name" : "王五", "age" : 22, "gender" : "女" }
```