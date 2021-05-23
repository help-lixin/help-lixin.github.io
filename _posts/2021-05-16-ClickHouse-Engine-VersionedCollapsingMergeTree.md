---
layout: post
title: 'ClickHouse VersionedCollapsingMergeTree表引擎(十)'
date: 2021-05-16
author: 李新
tags:  ClickHouse
---

### (1). VersionedCollapsingMergeTree介绍
> VersionedCollapsingMergeTree解决了CollapsingMergeTree乱序写入情况下无法正常折叠问题.  
> VersionedCollapsingMergeTree在建表时,要求增加长一个字段:verion.当主键相同,且version相同,sign相反的行,在分区合并时,进行删除.  

### (2). 案例一
```
# 1. 创建库
CREATE DATABASE test;

USE test;

 2. 创建表(需要指定:sign标记),注意:排序是:order_id和create_time
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
   sign               Int8,
   version            UInt8
) ENGINE = VersionedCollapsingMergeTree(sign,version)   
  PARTITION BY merchant_id
  ORDER BY (order_id,create_time);
 
 
# 3.插入数据
INSERT INTO t_order(order_id,order_no,customer_id,customer_name,customer_phone,customer_address,merchant_id,total_money,order_status,create_time,sign,version) 
VALUES (1,'000000000010',1000,'张三','13799999999','广东省深圳市南山区',7,500.50,'S','2021-05-16 12:12:12',1,1),
       (2,'000000000011',1001,'李四','137999999998','广东省广州市',8,600.50,'S','2020-04-01 11:11:11',1,1),
	   (3,'000000000012',1002,'赵六','137999999997','广东省珠海市',9,700.50,'E','2021-05-01 11:11:11',1,1);  


INSERT INTO t_order(order_id,order_no,customer_id,customer_name,customer_phone,customer_address,merchant_id,total_money,order_status,create_time,sign,version) 
VALUES (1,'000000000010',1000,'张三','13799999999','广东省深圳市南山区',7,500.50,'S','2021-05-18 18:18:18',1,2);

# 4. 未合并分区前,表数据状态(order_id=1的数据有两条)
SELECT * FROM t_order;
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address─┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┬─version─┐
│        2 │ 000000000011 │        1001 │ 李四          │ 137999999998   │ 广东省广州市     │           8 │    600.5000 │ 2020-04-01 11:11:11 │ S            │    1 │       1 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴──────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┴─────────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address───┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┬─version─┐
│        1 │ 000000000010 │        1000 │ 张三          │ 13799999999    │ 广东省深圳市南山区 │           7 │    500.5000 │ 2021-05-16 12:12:12 │ S            │    1 │       1 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴────────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┴─────────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address─┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┬─version─┐
│        3 │ 000000000012 │        1002 │ 赵六          │ 137999999997   │ 广东省珠海市     │           9 │    700.5000 │ 2021-05-01 11:11:11 │ E            │    1 │       1 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴──────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┴─────────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address───┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┬─version─┐
│        1 │ 000000000010 │        1000 │ 张三          │ 13799999999    │ 广东省深圳市南山区 │           7 │    500.5000 │ 2021-05-18 18:18:18 │ S            │    1 │       2 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴────────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┴─────────┘

# 5. 折叠指定版本的数据(7+1+2021-05-18 18:18:18)
INSERT INTO t_order(order_id,order_no,customer_id,customer_name,customer_phone,customer_address,merchant_id,total_money,order_status,create_time,sign,version) 
VALUES (1,'000000000010',1000,'张三','13799999999','广东省深圳市南山区',7,500.88,'S','2021-05-18 18:18:18',-1,2);


# 6. 未合并分区前,表数据状态(order_id=1的数据有三条,其中,sign=1的有一条)
SELECT * FROM t_order;
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address───┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┬─version─┐
│        1 │ 000000000010 │        1000 │ 张三          │ 13799999999    │ 广东省深圳市南山区 │           7 │    500.5000 │ 2021-05-16 12:12:12 │ S            │    1 │       1 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴────────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┴─────────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address───┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┬─version─┐
│        1 │ 000000000010 │        1000 │ 张三          │ 13799999999    │ 广东省深圳市南山区 │           7 │    500.5000 │ 2021-05-18 18:18:18 │ S            │    1 │       2 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴────────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┴─────────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address─┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┬─version─┐
│        2 │ 000000000011 │        1001 │ 李四          │ 137999999998   │ 广东省广州市     │           8 │    600.5000 │ 2020-04-01 11:11:11 │ S            │    1 │       1 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴──────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┴─────────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address─┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┬─version─┐
│        3 │ 000000000012 │        1002 │ 赵六          │ 137999999997   │ 广东省珠海市     │           9 │    700.5000 │ 2021-05-01 11:11:11 │ E            │    1 │       1 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴──────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┴─────────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address───┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┬─version─┐
│        1 │ 000000000010 │        1000 │ 张三          │ 13799999999    │ 广东省深圳市南山区 │           7 │    500.8800 │ 2021-05-18 18:18:18 │ S            │   -1 │       2 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴────────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┴─────────┘

# 7. 优化表
OPTIMIZE TABLE t_order;
Ok.


# 8. 折叠表后效果(删除掉了最近的那条数据)
SELECT * FROM t_order;
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address─┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┬─version─┐
│        2 │ 000000000011 │        1001 │ 李四          │ 137999999998   │ 广东省广州市     │           8 │    600.5000 │ 2020-04-01 11:11:11 │ S            │    1 │       1 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴──────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┴─────────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address─┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┬─version─┐
│        3 │ 000000000012 │        1002 │ 赵六          │ 137999999997   │ 广东省珠海市     │           9 │    700.5000 │ 2021-05-01 11:11:11 │ E            │    1 │       1 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴──────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┴─────────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address───┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┬─sign─┬─version─┐
│        1 │ 000000000010 │        1000 │ 张三          │ 13799999999    │ 广东省深圳市南山区 │           7 │    500.5000 │ 2021-05-16 12:12:12 │ S            │    1 │       1 │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴────────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┴──────┴─────────┘
```
### (3). 总结
> VersionedCollapsingMergeTree与CollapsingMergeTree的不同在于:可以指定折叠哪行数据.    