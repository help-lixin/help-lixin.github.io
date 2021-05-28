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
> update默认情况下,只会修入一条数据(可通过参数配置multi:true),而,updateMany则可以修改多条数据.  

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

### 8. MongoDB索引
```
# 1. 创建新的库
> use test2;
switched to db test2

# 2.创建Collection,并批量插入20条数据
> db.orders.insert([{order_id:1,order_no:"000000001",customer_id:1,customer_name:"张三",customer_phone:"13777777771",merchant_id:8888,total_money:10.58},{order_id:2,order_no:"000000002",customer_id:2,customer_name:"李四",customer_phone:"13777777772",merchant_id:8888,total_money:11.58},{order_id:3,order_no:"000000003",customer_id:3,customer_name:"王五",customer_phone:"13777777773",merchant_id:8888,total_money:12.58},{order_id:4,order_no:"000000004",customer_id:4,customer_name:"赵六",customer_phone:"13777777774",merchant_id:8888,total_money:13.58},{order_id:5,order_no:"000000005",customer_id:5,customer_name:"小何",customer_phone:"13777777775",merchant_id:8888,total_money:14.58},{order_id:6,order_no:"000000006",customer_id:6,customer_name:"小李",customer_phone:"13777777776",merchant_id:8888,total_money:15.58},{order_id:7,order_no:"000000007",customer_id:7,customer_name:"小娜",customer_phone:"13777777777",merchant_id:8888,total_money:16.58},{order_id:8,order_no:"000000008",customer_id:8,customer_name:"小丽",customer_phone:"137777777778",merchant_id:8888,total_money:17.58},{order_id:9,order_no:"000000009",customer_id:9,customer_name:"小赵",customer_phone:"13777777779",merchant_id:8888,total_money:18.58},{order_id:10,order_no:"0000000010",customer_id:10,customer_name:"小聪",customer_phone:"13777777710",merchant_id:8888,total_money:19.58},{order_id:11,order_no:"0000000011",customer_id:1,customer_name:"张三",customer_phone:"13777777771",merchant_id:9999,total_money:22.58},{order_id:12,order_no:"0000000012",customer_id:2,customer_name:"李四",customer_phone:"13777777772",merchant_id:9999,total_money:23.58},{order_id:13,order_no:"0000000013",customer_id:3,customer_name:"王五",customer_phone:"13777777773",merchant_id:9999,total_money:24.58},{order_id:14,order_no:"0000000014",customer_id:4,customer_name:"赵六",customer_phone:"13777777774",merchant_id:9999,total_money:25.58},{order_id:15,order_no:"0000000015",customer_id:5,customer_name:"小何",customer_phone:"13777777775",merchant_id:9999,total_money:26.58},{order_id:16,order_no:"0000000016",customer_id:6,customer_name:"小李",customer_phone:"13777777776",merchant_id:9999,total_money:27.58},{order_id:17,order_no:"0000000017",customer_id:7,customer_name:"小娜",customer_phone:"13777777777",merchant_id:9999,total_money:28.58},{order_id:18,order_no:"0000000018",customer_id:8,customer_name:"小丽",customer_phone:"137777777778",merchant_id:9999,total_money:29.58},{order_id:19,order_no:"0000000019",customer_id:9,customer_name:"小赵",customer_phone:"13777777779",merchant_id:9999,total_money:30.58},{order_id:20,order_no:"0000000020",customer_id:10,customer_name:"小聪",customer_phone:"13777777710",merchant_id:9999,total_money:31.58}]);

# 3. 查看Collection(orders)中的所有索引.  
> db.orders.getIndexes();
[ 
	{ 
		"v" : 2,                  # 索引引擎的版本号
		"key" : { "_id" : 1 },    # 主键索引:_id升序索引
		"name" : "_id_"           # 索引别名
	} 
]


# 4. 创建索引(以租户ID升序和订单号降序,创建索引)
# 语法中Key值为你要创建的索引字段:1为指定按升序创建索引.
# 如果你想按降序来创建索引指定为:-1.
# background : 以后台方式创建索引(默认:false).
# unique     : 建立的索引是否唯一(默认:false).
# name       : 索引名称(可选)
# > db.orders.createIndex({order_id:-1},{background:true,unique:true});
# > db.orders.createIndex({merchant_id:1,order_id:-1},{background:true});

> db.orders.createIndex({merchant_id:1,order_no:-1},{background:true});
{
	"createdCollectionAutomatically" : false,
	"numIndexesBefore" : 1,
	"numIndexesAfter" : 2,
	"ok" : 1
}

# 5. 查看Colletion(order)所有的索引
> db.orders.getIndexes();
[
	{
		"v" : 2,
		"key" : {
			"_id" : 1
		},
		"name" : "_id_"
	},
	{
		"v" : 2,
		"key" : {
			"merchant_id" : 1,
			"order_no" : -1
		},
		"name" : "merchant_id_1_order_no_-1",
		"background" : true
	}
]

# 6. 按名称删除索引
> db.orders.dropIndex("merchant_id_1_order_no_-1")
{ "nIndexesWas" : 2, "ok" : 1 }

```
### 9. MongoDB 执行计划
```
# 1. 查看执行计划
> db.orders.find({merchant_id:8888}).explain();
{
	"queryPlanner" : {
		"plannerVersion" : 1,
		"namespace" : "test2.orders",
		"indexFilterSet" : false,
		"parsedQuery" : {
			"merchant_id" : {
				"$eq" : 8888
			}
		},
		"queryHash" : "0A2C21EA",
		"planCacheKey" : "C7F0160F",
		"winningPlan" : {
			"stage" : "FETCH",                                  #  COLLSCAN:全集合扫描/FETCH:抓取
			"inputStage" : {
				"stage" : "IXSCAN",
				"keyPattern" : {
					"merchant_id" : 1,
					"order_no" : -1
				},
				"indexName" : "merchant_id_1_order_no_-1",       # 使用的索引名称
				"isMultiKey" : false,
				"multiKeyPaths" : {
					"merchant_id" : [ ],
					"order_no" : [ ]
				},
				"isUnique" : false,
				"isSparse" : false,
				"isPartial" : false,
				"indexVersion" : 2,
				"direction" : "forward",
				"indexBounds" : {
					"merchant_id" : [
						"[8888.0, 8888.0]"
					],
					"order_no" : [
						"[MaxKey, MinKey]"
					]
				}
			}
		},
		"rejectedPlans" : [ ]
	},
	"serverInfo" : {
		"host" : "lixin-macbook.local",
		"port" : 27017,
		"version" : "4.4.1",
		"gitVersion" : "ad91a93a5a31e175f5cbf8c69561e788bbc55ce1"
	},
	"ok" : 1

# 2. 类似MySQL索引覆盖(查询的结果集,来自于索引时,会只读取索引,无须回行读取数据行)
> db.orders.find({merchant_id:8888},{merchant_id:1,_id:0}).explain();
{
	"queryPlanner" : {
		"plannerVersion" : 1,
		"namespace" : "test2.orders",
		"indexFilterSet" : false,
		"parsedQuery" : {
			"merchant_id" : {
				"$eq" : 8888
			}
		},
		"queryHash" : "E87DFABE",
		"planCacheKey" : "8FF1E7F9",
		"winningPlan" : {
			"stage" : "PROJECTION_SIMPLE",
			"transformBy" : {
				"merchant_id" : 1
			},
			"inputStage" : {
				"stage" : "FETCH",
				"inputStage" : {
					"stage" : "IXSCAN",
					"keyPattern" : {
						"merchant_id" : 1,
						"order_no" : -1
					},
					"indexName" : "merchant_id_1_order_no_-1",
					"isMultiKey" : false,
					"multiKeyPaths" : {
						"merchant_id" : [ ],
						"order_no" : [ ]
					},
					"isUnique" : false,
					"isSparse" : false,
					"isPartial" : false,
					"indexVersion" : 2,
					"direction" : "forward",
					"indexBounds" : {
						"merchant_id" : [
							"[8888.0, 8888.0]"
						],
						"order_no" : [
							"[MaxKey, MinKey]"
						]
					}
				}
			}
		},
		"rejectedPlans" : [ ]
	},
	"serverInfo" : {
		"host" : "lixin-macbook.local",
		"port" : 27017,
		"version" : "4.4.1",
		"gitVersion" : "ad91a93a5a31e175f5cbf8c69561e788bbc55ce1"
	},
	"ok" : 1
}
```