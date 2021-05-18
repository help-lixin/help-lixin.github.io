---
layout: post
title: 'ClickHouse 基本数据类型(三)'
date: 2021-05-16
author: 李新
tags:  ClickHouse
---

### (1). ClickHouse支持的数据类型
["ClickHouse支持的数据类型参考"](https://clickhouse.tech/docs/en/sql-reference/data-types)

### (2). ClickHouse基本数据类型
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
  age  UInt8,
  sex Enum('M' = 1 , 'F' = 2),
  phone Tuple(String, String),
  address Array(String)
) ENGINE = Memory();

# 3. 插入数据
INSERT INTO t_users(id,name,age,sex,phone,address) VALUES(1,'张三',20,'M',('湖南','13788888881'),['广东','湖南']);
INSERT INTO t_users(id,name,age,sex,phone,address) VALUES(2,'李四',21,'F',('湖北','13788888882'),['河南','湖北']);
INSERT INTO t_users(id,name,age,sex,phone,address) VALUES(3,'王五',22,'M',('福建','13788888883'),['河北','福建']);
INSERT INTO t_users(id,name,age,sex,phone,address) VALUES(4,'赵六',23,'F',('重庆','13788888884'),['北京','重庆']);

# 4. 数组检索
SELECT 
  *,
   address[2] AS address_2
FROM t_users;


# 5. 枚举检索
SELECT * 
FROM t_users 
WHERE sex = 'M';


# 6. Tuple检索
SELECT 
  id,
  name , 
  phone.1  as phone_addr,
  phone.2  as phone_num
FROM t_users WHERE phone.1 = '湖南';
```

### (3). ClickHouse嵌套数据类型

```
# 1. 创建订单与订单详细(嵌套表)
CREATE TABLE t_order (
  order_id        Int32,
  total_memory    Decimal(9, 4),
  OrderDetail Nested(
      order_detail_id Int32,
	  goods_name      String,
	  goods_price     Decimal(9, 4)
  )
) ENGINE = Memory();


#  2. 查看表结构
9e40ca366829 :) DESC t_order;
┌─name────────────────────────┬─type─────────────────┬─default_type─┬─default_expression─┬─comment─┬─codec_expression─┬─ttl_expression─┐
│ order_id                    │ Int32                │              │                    │         │                  │                │
│ total_memory                │ Decimal(9, 4)        │              │                    │         │                  │                │
│ OrderDetail.order_detail_id │ Array(Int32)         │              │                    │         │                  │                │
│ OrderDetail.goods_name      │ Array(String)        │              │                    │         │                  │                │
│ OrderDetail.goods_price     │ Array(Decimal(9, 4)) │              │                    │         │                  │                │
└─────────────────────────────┴──────────────────────┴──────────────┴────────────────────┴─────────┴──────────────────┴────────────────┘

# 3. 插入数据
INSERT INTO t_order(order_id,total_memory,OrderDetail.order_detail_id,OrderDetail.goods_name,OrderDetail.goods_price) VALUES(1,200,[1,2],['test1','test2'],[100,100]);
INSERT INTO t_order(order_id,total_memory,OrderDetail.order_detail_id,OrderDetail.goods_name,OrderDetail.goods_price) VALUES(2,500,[1],['test3'],[500]);


# 4. 检索所有数据
SELECT 
  *
FROM t_order

┌─order_id─┬─total_memory─┬─OrderDetail.order_detail_id─┬─OrderDetail.goods_name─┬─OrderDetail.goods_price─┐
│        1 │     200.0000 │ [1,2]                       │ ['test1','test2']      │ [100.0000,100.0000]     │
└──────────┴──────────────┴─────────────────────────────┴────────────────────────┴─────────────────────────┘
┌─order_id─┬─total_memory─┬─OrderDetail.order_detail_id─┬─OrderDetail.goods_name─┬─OrderDetail.goods_price─┐
│        2 │     500.0000 │ [1]                         │ ['test3']              │ [500.0000]              │
└──────────┴──────────────┴─────────────────────────────┴────────────────────────┴─────────────────────────┘


# 5. 检索(通过ARRAY JOIN关键字可以把nested字段的数组由行转列)
SELECT 
  *  
FROM t_order  
ARRAY JOIN OrderDetail 
WHERE OrderDetail.goods_name = 'test1';

┌─order_id─┬─total_memory─┬─OrderDetail.order_detail_id─┬─OrderDetail.goods_name─┬─OrderDetail.goods_price─┐
│        1 │     200.0000 │                           1 │ test1                  │                100.0000 │
└──────────┴──────────────┴─────────────────────────────┴────────────────────────┴─────────────────────────┘
```
