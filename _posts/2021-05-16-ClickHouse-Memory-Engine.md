---
layout: post
title: 'ClickHouse Memory表引擎(三)'
date: 2021-05-16
author: 李新
tags:  ClickHouse
---

### (1). ClickHouse支持的数据类型
["ClickHouse支持的数据类型介绍"](https://clickhouse.tech/docs/zh/sql-reference/data-types/)

### (2). ClickHouse Memory案例
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