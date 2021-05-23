---
layout: post
title: 'ClickHouse CollapsingMergeTree表引擎(九)'
date: 2021-05-16
author: 李新
tags:  ClickHouse
---

### (1). CollapsingMergeTree介绍
> CollapsingMergeTree就是一种通过以增代删的思路,支持行级数据修改和删除的表引擎.它通过定义一个sign标记位字段,记录数据行的状态.     
> 如果sign标记为1,则表示这是一行有效的数据.如果sign标记为-1,则表示这行数据需要被删除.当CollapsingMergeTree分区合并时,同一数据分区内,sign标记为1和-1的一组数据会被抵消删除.   
> 每次需要新增数据时,写入一行sign标记为1的数据.需要删除数据时,则写入一行sign标记为-1的数据. 

### (2). 案例一
```
# 1. 创建库
CREATE DATABASE test;

USE test;

 2. 创建表(需要指定:sign标记)
CREATE TABLE IF NOT EXISTS t_order (
   order_id          Int32,
   order_no          String,
   customer_id       Int32,
   customer_name     String,
   customer_phone    String,
   customer_address  String,
   merchant_id        Int32,
   total_money        Decimal(9, 4),
   create_time        DateTime,
   order_status       Enum('S' = 1 , 'E' = 2),
   sign               Int8
) ENGINE = CollapsingMergeTree(sign)   
  PARTITION BY merchant_id
  ORDER BY (order_id,create_time);

# 3.插入数据
INSERT INTO t_order(order_id,order_no,customer_id,customer_name,customer_phone,customer_address,merchant_id,total_money,order_status,create_time,sign) 
VALUES (1,'000000000010',1000,'张三','13799999999','广东省深圳市南山区',7,500.50,'S','2021-05-16 12:12:12',1),
       (2,'000000000011',1001,'李四','137999999998','广东省广州市',8,600.50,'S','2020-04-01 11:11:11',1),
	   (3,'000000000012',1002,'赵六','137999999997','广东省珠海市',9,700.50,'E','2021-05-01 11:11:11',1);  

# 4. 检索数据
SELECT * FROM t_order;
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address─┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┐
│        2 │ 000000000011 │        1001 │ 李四          │ 137999999998   │ 广东省广州市     │           8 │    600.5000 │ 2020-04-01 11:11:11 │ S            │    1 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴──────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address───┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┐
│        1 │ 000000000010 │        1000 │ 张三          │ 13799999999    │ 广东省深圳市南山区 │           7 │    500.5000 │ 2021-05-16 12:12:12 │ S            │    1 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴────────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address─┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┐
│        3 │ 000000000012 │        1002 │ 赵六          │ 137999999997   │ 广东省珠海市     │           9 │    700.5000 │ 2021-05-01 11:11:11 │ E            │    1 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴──────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┘

# 5. 插入一条sign=-1的数据(注意:我这里调整了时间)
INSERT INTO t_order(order_id,order_no,customer_id,customer_name,customer_phone,customer_address,merchant_id,total_money,order_status,create_time,sign) 
VALUES (1,'000000000010',1000,'张三','13799999999','广东省深圳市南山区',7,500.80,'E','2021-05-16 12:18:18',-1);


# 6. 未合并分区前的数据(order_id有2条).
SELECT * FROM t_order;
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address─┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┐
│        2 │ 000000000011 │        1001 │ 李四          │ 137999999998   │ 广东省广州市     │           8 │    600.5000 │ 2020-04-01 11:11:11 │ S            │    1 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴──────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address───┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┐
│        1 │ 000000000010 │        1000 │ 张三          │ 13799999999    │ 广东省深圳市南山区 │           7 │    500.5000 │ 2021-05-16 12:12:12 │ S            │    1 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴────────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address─┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┐
│        3 │ 000000000012 │        1002 │ 赵六          │ 137999999997   │ 广东省珠海市     │           9 │    700.5000 │ 2021-05-01 11:11:11 │ E            │    1 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴──────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address───┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┐
│        1 │ 000000000010 │        1000 │ 张三          │ 13799999999    │ 广东省深圳市南山区 │           7 │    500.8000 │ 2021-05-16 12:18:18 │ E            │   -1 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴────────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┘

# 7. 优化表(合并分区)
OPTIMIZE TABLE t_order;
Ok.

# 8. 校验数据
#  为什么?没有进行折叠?因为,折叠实际上是受:order by (order_id,crate_time)影响的
#  折叠条件:merchant_id+order_id+create_time,只有这三者达到唯一才能进行折叠,折叠与主键无关来着的.
SELECT * FROM t_order　;
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address─┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┐
│        2 │ 000000000011 │        1001 │ 李四          │ 137999999998   │ 广东省广州市     │           8 │    600.5000 │ 2020-04-01 11:11:11 │ S            │    1 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴──────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address─┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┐
│        3 │ 000000000012 │        1002 │ 赵六          │ 137999999997   │ 广东省珠海市     │           9 │    700.5000 │ 2021-05-01 11:11:11 │ E            │    1 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴──────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address───┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┐
│        1 │ 000000000010 │        1000 │ 张三          │ 13799999999    │ 广东省深圳市南山区 │           7 │    500.5000 │ 2021-05-16 12:12:12 │ S            │    1 │
│        1 │ 000000000010 │        1000 │ 张三          │ 13799999999    │ 广东省深圳市南山区 │           7 │    500.8000 │ 2021-05-16 12:18:18 │ E            │   -1 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴────────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┘
```

### (3). 案例二
```
# 1. 创建库
CREATE DATABASE test;

USE test;

 2. 创建表(需要指定:sign标记)
CREATE TABLE IF NOT EXISTS t_order (
   order_id          Int32,
   order_no          String,
   customer_id       Int32,
   customer_name     String,
   customer_phone    String,
   customer_address  String,
   merchant_id        Int32,
   total_money        Decimal(9, 4),
   create_time        DateTime,
   order_status       Enum('S' = 1 , 'E' = 2),
   sign               Int8
) ENGINE = CollapsingMergeTree(sign)   
  PARTITION BY merchant_id
  ORDER BY (order_id,create_time);

# 3.插入数据
INSERT INTO t_order(order_id,order_no,customer_id,customer_name,customer_phone,customer_address,merchant_id,total_money,order_status,create_time,sign) 
VALUES (1,'000000000010',1000,'张三','13799999999','广东省深圳市南山区',7,500.50,'S','2021-05-16 12:12:12',1),
       (2,'000000000011',1001,'李四','137999999998','广东省广州市',8,600.50,'S','2020-04-01 11:11:11',1),
	   (3,'000000000012',1002,'赵六','137999999997','广东省珠海市',9,700.50,'E','2021-05-01 11:11:11',1);  

# 4. 检索数据
SELECT * FROM t_order;
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address─┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┐
│        2 │ 000000000011 │        1001 │ 李四          │ 137999999998   │ 广东省广州市     │           8 │    600.5000 │ 2020-04-01 11:11:11 │ S            │    1 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴──────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address───┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┐
│        1 │ 000000000010 │        1000 │ 张三          │ 13799999999    │ 广东省深圳市南山区 │           7 │    500.5000 │ 2021-05-16 12:12:12 │ S            │    1 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴────────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address─┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┐
│        3 │ 000000000012 │        1002 │ 赵六          │ 137999999997   │ 广东省珠海市     │           9 │    700.5000 │ 2021-05-01 11:11:11 │ E            │    1 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴──────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┘

# 5. 插入一条sign=-1的数据(注意:我这里调整了时间)
INSERT INTO t_order(order_id,order_no,customer_id,customer_name,customer_phone,customer_address,merchant_id,total_money,order_status,create_time,sign) 
VALUES (1,'000000000010',1000,'张三','13799999999','广东省深圳市南山区',7,500.80,'E','2021-05-16 12:12:12',-1);


# 6. 未合并分区前的数据(order_id有2条).
SELECT * FROM t_order;
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address─┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┐
│        2 │ 000000000011 │        1001 │ 李四          │ 137999999998   │ 广东省广州市     │           8 │    600.5000 │ 2020-04-01 11:11:11 │ S            │    1 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴──────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address───┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┐
│        1 │ 000000000010 │        1000 │ 张三          │ 13799999999    │ 广东省深圳市南山区 │           7 │    500.5000 │ 2021-05-16 12:12:12 │ S            │    1 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴────────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address─┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┐
│        3 │ 000000000012 │        1002 │ 赵六          │ 137999999997   │ 广东省珠海市     │           9 │    700.5000 │ 2021-05-01 11:11:11 │ E            │    1 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴──────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address───┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┐
│        1 │ 000000000010 │        1000 │ 张三          │ 13799999999    │ 广东省深圳市南山区 │           7 │    500.8000 │ 2021-05-16 12:18:18 │ E            │   -1 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴────────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┘

# 7. 优化表(合并分区)
OPTIMIZE TABLE t_order;
Ok.

# 8. 合并分区后的数据(order_id=1的数据被删除了).
9e40ca366829 :) SELECT * FROM t_order;
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address─┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┐
│        3 │ 000000000012 │        1002 │ 赵六          │ 137999999997   │ 广东省珠海市     │           9 │    700.5000 │ 2021-05-01 11:11:11 │ E            │    1 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴──────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address─┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┐
│        2 │ 000000000011 │        1001 │ 李四          │ 137999999998   │ 广东省广州市     │           8 │    600.5000 │ 2020-04-01 11:11:11 │ S            │    1 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴──────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┘
```

### (4). 案例三
```
# 1. 创建库
CREATE DATABASE test;

USE test;

 2. 创建表(需要指定:sign标记)
CREATE TABLE IF NOT EXISTS t_order (
   order_id          Int32,
   order_no          String,
   customer_id       Int32,
   customer_name     String,
   customer_phone    String,
   customer_address  String,
   merchant_id        Int32,
   total_money        Decimal(9, 4),
   create_time        DateTime,
   order_status       Enum('S' = 1 , 'E' = 2),
   sign               Int8
) ENGINE = CollapsingMergeTree(sign)   
  PARTITION BY merchant_id
  ORDER BY order_id;

# 3.插入数据
INSERT INTO t_order(order_id,order_no,customer_id,customer_name,customer_phone,customer_address,merchant_id,total_money,order_status,create_time,sign) 
VALUES (1,'000000000010',1000,'张三','13799999999','广东省深圳市南山区',7,500.50,'S','2021-05-16 12:12:12',1),
       (2,'000000000011',1001,'李四','137999999998','广东省广州市',8,600.50,'S','2020-04-01 11:11:11',1),
	   (3,'000000000012',1002,'赵六','137999999997','广东省珠海市',9,700.50,'E','2021-05-01 11:11:11',1);  

INSERT INTO t_order(order_id,order_no,customer_id,customer_name,customer_phone,customer_address,merchant_id,total_money,order_status,create_time,sign) 
VALUES (1,'000000000010',1000,'张三','13799999999','广东省深圳市南山区',7,500.50,'E','2021-05-18 18:18:18',1);



# 4. 检索数据
SELECT * FROM t_order;
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address───┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┐
│        1 │ 000000000010 │        1000 │ 张三          │ 13799999999    │ 广东省深圳市南山区 │           7 │    500.5000 │ 2021-05-16 12:12:12 │ S            │    1 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴────────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address─┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┐
│        2 │ 000000000011 │        1001 │ 李四          │ 137999999998   │ 广东省广州市     │           8 │    600.5000 │ 2020-04-01 11:11:11 │ S            │    1 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴──────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address───┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┐
│        1 │ 000000000010 │        1000 │ 张三          │ 13799999999    │ 广东省深圳市南山区 │           7 │    500.5000 │ 2021-05-18 18:18:18 │ E            │    1 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴────────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address─┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┐
│        3 │ 000000000012 │        1002 │ 赵六          │ 137999999997   │ 广东省珠海市     │           9 │    700.5000 │ 2021-05-01 11:11:11 │ E            │    1 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴──────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┘

# 5. 插入一条sign=-1的数据(注意:我这里调整了时间)
INSERT INTO t_order(order_id,order_no,customer_id,customer_name,customer_phone,customer_address,merchant_id,total_money,order_status,create_time,sign) 
VALUES (1,'000000000010',1000,'张三','13799999999','广东省深圳市南山区',7,500.80,'E','2021-05-16 12:12:12',-1);

# 6. 未优化表之前的数据(order_id=1的数有三条)
SELECT * FROM t_order;
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address───┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┐
│        1 │ 000000000010 │        1000 │ 张三          │ 13799999999    │ 广东省深圳市南山区 │           7 │    500.5000 │ 2021-05-16 12:12:12 │ S            │    1 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴────────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address───┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┐
│        1 │ 000000000010 │        1000 │ 张三          │ 13799999999    │ 广东省深圳市南山区 │           7 │    500.5000 │ 2021-05-18 18:18:18 │ E            │    1 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴────────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address─┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┐
│        2 │ 000000000011 │        1001 │ 李四          │ 137999999998   │ 广东省广州市     │           8 │    600.5000 │ 2020-04-01 11:11:11 │ S            │    1 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴──────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address───┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┐
│        1 │ 000000000010 │        1000 │ 张三          │ 13799999999    │ 广东省深圳市南山区 │           7 │    500.8000 │ 2021-05-16 12:12:12 │ E            │   -1 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴────────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address─┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┐
│        3 │ 000000000012 │        1002 │ 赵六          │ 137999999997   │ 广东省珠海市     │           9 │    700.5000 │ 2021-05-01 11:11:11 │ E            │    1 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴──────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┘

# 7. 优化表
OPTIMIZE TABLE t_order;


# 8. 优化表之后的数据
# 结论:删除了最早的一条数据
SELECT * FROM t_order;
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address─┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┐
│        2 │ 000000000011 │        1001 │ 李四          │ 137999999998   │ 广东省广州市     │           8 │    600.5000 │ 2020-04-01 11:11:11 │ S            │    1 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴──────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address─┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┐
│        3 │ 000000000012 │        1002 │ 赵六          │ 137999999997   │ 广东省珠海市     │           9 │    700.5000 │ 2021-05-01 11:11:11 │ E            │    1 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴──────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address───┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┐
│        1 │ 000000000010 │        1000 │ 张三          │ 13799999999    │ 广东省深圳市南山区 │           7 │    500.5000 │ 2021-05-18 18:18:18 │ E            │    1 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴────────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┘
```
### (5). 总结
> CollapsingMergeTree对数据折叠的条件是:分区+排序字段作为条件(唯一),才会对数据进行折叠(删除最早的一条数据).  