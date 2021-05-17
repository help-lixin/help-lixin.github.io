---
layout: post
title: 'ClickHouse 表引擎(三)'
date: 2021-05-16
author: 李新
tags:  ClickHouse
---

### (1). 表引擎
表引擎(即表的类型)决定了:  

* 数据的存储方式和位置,写到哪里以及从哪里读取数据.  
* 支持哪些查询以及如何支持. 
* 并发数据访问. 
* 索引的使用. 
* 是否可以执行多线程请求. 
* 数据复制参数.  

### (2). 引擎类型
+ MergeTree
适用于高负载任务的最通用和功能最强大的表引擎.这些引擎的共同特点是可以快速插入数据并进行后续的后台数据处理.MergeTree系列引擎支持数据复制(使用Replicated* 的引擎版本),分区和一些其他引擎不支持的其他功能.  
该类型的引擎：
- MergeTree
- ReplacingMergeTree
- SummingMergeTree
- AggregatingMergeTree
- CollapsingMergeTree
- VersionedCollapsingMergeTree
- GraphiteMergeTree

+ 日志
具有最小功能的轻量级引擎,当您需要快速写入许多小表(最多约100万行)并在以后整体读取它们时,该类型的引擎是最有效的. 
该类型的引擎：
- TinyLog
- StripeLog
- Log

+ 集成引擎
用于与其他的数据存储与处理系统集成的引擎.
该类型的引擎:
- Kafka
- MySQL
- ODBC
- JDBC
- HDFS

+ 其他引擎
- Distributed
- MaterializedView
- Dictionary
- Merge
- File
- Null
- Set
- Join
- URL
- View
- Memory
- Buffer

### (3). ClickHouse支持的数据类型
["ClickHouse支持的数据类型"](https://clickhouse.tech/docs/zh/sql-reference/data-types/)

### (4). ClickHouse Memory案例
> 需要注意:
> 1. ClickHouser的数据类型名称的大小写都必须与定义对应(Int32不能写成INT32).    
> 2. ClickHouse的引擎名称也要遵循大小写(Memory不能写成:memory).  
> 3. ClickHouse插入数据时,只能是单引号,不能写成双引号.  

```
# 1. 创建库
CREATE DATABASE IF NOT EXISTS test;
USE test;

# 2. 创建表
CREATE TABLE t_users(
   id Int32,
   name   String,
   age  UInt8
) ENGINE = Memory();

# 3. 插入数据
INSERT INTO t_users(id,name,age) VALUES(1,'张三',20);
INSERT INTO t_users(id,name,age) VALUES(2,'李四',21);
INSERT INTO t_users(id,name,age) VALUES(3,'王五',22);
INSERT INTO t_users(id,name,age) VALUES(4,'赵六',23);
```
