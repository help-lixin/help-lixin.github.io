---
layout: post
title: 'ClickHouse Log表引擎(四)'
date: 2021-05-16
author: 李新
tags:  ClickHouse
---

### (1). TinyLog 表引擎测试
> 仅适用于:读多,少写的场景.   
> 1. 存在磁盘中.     
> 2. 不支持索引.      
> 3. 不支持并发控制.   

```
# 1. 创建库
CREATE DATABASE test2;

USE test2;

# 2. 创建表
CREATE TABLE IF NOT EXISTS t_order (
   order_id          Int32,
   order_no          String,
   customer_id       Int32,
   customer_name     String,
   customer_phone    String,
   customer_address  String,
   merchant_id        Int32,
   total_money        Decimal(9, 4),
   order_status       Enum('S' = 1 , 'E' = 2)
) ENGINE = TinyLog();

# 3. 插入数据
INSERT INTO t_order(order_id,order_no,customer_id,customer_name,customer_phone,customer_address,merchant_id,total_money,order_status) 
VALUES(1,'000000000010',1000,'张三','13799999999','广东省深圳市南山区',7,500.50,'S');
INSERT INTO t_order(order_id,order_no,customer_id,customer_name,customer_phone,customer_address,merchant_id,total_money,order_status) 
VALUES(2,'000000000011',1001,'李四','137999999998','广东省广州市',8,600.50,'S');


# 4. 检索数据(TinyLog与Log的不同之处在于,Log会根据CPU对文件进行分片,而TinyLog则不会对数据进行分片)
9e40ca366829 :) SELECT * FROM t_order;
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address───┬─merchant_id─┬─total_money─┬─order_status─┐
│        1 │ 000000000010 │        1000 │ 张三          │ 13799999999    │ 广东省深圳市南山区 │           7 │    500.5000 │ S            │
│        2 │ 000000000011 │        1001 │ 李四          │ 137999999998   │ 广东省广州市       │           8 │    600.5000 │ S            │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴────────────────────┴─────────────┴─────────────┴──────────────┘

# 5. 总结
# TinyLog是本地存储数据(/var/lib/clickhouse/data),所有的数据由于没有分块,所以不支持并发(读/写)处理.  

# test2对应的是数据库
# t_order对应的是表
root@9e40ca366829:/var/lib/clickhouse/data/test2/t_order# pwd
/var/lib/clickhouse/data/test2/t_order

# 在表里的每一列,都对应一个物理文件.
root@9e40ca366829:/var/lib/clickhouse/data/test2/t_order# ll
total 40
drwxr-x--- 12 root root 384 May 17 12:38 ./
drwxr-x---  3 root root  96 May 17 12:36 ../
-rw-r-----  1 root root 101 May 17 12:38 customer_address.bin
-rw-r-----  1 root root  60 May 17 12:38 customer_id.bin
-rw-r-----  1 root root  66 May 17 12:38 customer_name.bin
-rw-r-----  1 root root  77 May 17 12:38 customer_phone.bin
-rw-r-----  1 root root  60 May 17 12:38 merchant_id.bin
-rw-r-----  1 root root  60 May 17 12:38 order_id.bin
-rw-r-----  1 root root  78 May 17 12:38 order_no.bin
-rw-r-----  1 root root  54 May 17 12:38 order_status.bin
-rw-r-----  1 root root 324 May 17 12:38 sizes.json
-rw-r-----  1 root root  60 May 17 12:38 total_money.bin

# 元数据存储(/var/lib/clickhouse/metadata)
root@9e40ca366829:/var/lib/clickhouse/metadata/test2# pwd
/var/lib/clickhouse/metadata/test2

# t_order.sql存储的是建表的脚本.
root@9e40ca366829:/var/lib/clickhouse/metadata/test2# ll
total 4
drwxr-x--- 3 root root  96 May 17 12:36 ./
drwxr-x--- 3 root root  96 May 17 12:36 ../
-rw-r----- 1 root root 339 May 17 12:36 t_order.sql

# 查看sql文件.
root@9e40ca366829:/var/lib/clickhouse/metadata/test2# cat t_order.sql
ATTACH TABLE _ UUID '3b795b03-d582-4da2-be8f-5f208696cfe3'
(
    `order_id` Int32,
    `order_no` String,
    `customer_id` Int32,
    `customer_name` String,
    `customer_phone` String,
    `customer_address` String,
    `merchant_id` Int32,
    `total_money` Decimal(9, 4),
    `order_status` Enum8('S' = 1, 'E' = 2)
)
ENGINE = TinyLog
```
### (2). Log表引擎测试
```
# 1. 创建库
CREATE DATABASE test3;

USE test3;

# 2. 创建表
CREATE TABLE IF NOT EXISTS t_order (
   order_id          Int32  COMMENT '订单ID',
   order_no          String  COMMENT '订单编号',
   customer_id       Int32  COMMENT '客户ID',
   customer_name     String  COMMENT '客户名称',
   customer_phone    String  COMMENT '客户手机号码',
   customer_address  String  COMMENT '客户地址',
   merchant_id        Int32  COMMENT '商户ID',
   total_money        Decimal(9, 4)  COMMENT '订单总金额',
   order_status       Enum('S' = 1 , 'E' = 2)  COMMENT '订单状态'
) ENGINE = Log();

# 3. 插入数据
INSERT INTO t_order(order_id,order_no,customer_id,customer_name,customer_phone,customer_address,merchant_id,total_money,order_status) 
VALUES(1,'000000000010',1000,'张三','13799999999','广东省深圳市南山区',7,500.50,'S');
INSERT INTO t_order(order_id,order_no,customer_id,customer_name,customer_phone,customer_address,merchant_id,total_money,order_status) 
VALUES(2,'000000000011',1001,'李四','137999999998','广东省广州市',8,600.50,'S');


# 4. 检索数据(与TinyLog不同,数据是会分布存储的.)
9e40ca366829 :) SELECT * FROM t_order;

┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address───┬─merchant_id─┬─total_money─┬─order_status─┐
│        1 │ 000000000010 │        1000 │ 张三          │ 13799999999    │ 广东省深圳市南山区 │           7 │    500.5000 │ S            │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴────────────────────┴─────────────┴─────────────┴──────────────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address─┬─merchant_id─┬─total_money─┬─order_status─┐
│        2 │ 000000000011 │        1001 │ 李四          │ 137999999998   │ 广东省广州市     │           8 │    600.5000 │ S            │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴──────────────────┴─────────────┴─────────────┴──────────────┘

# 数据存放目录(/var/lib/clickhouse/data)
root@9e40ca366829:/var/lib/clickhouse/data/test3/t_order# pwd
/var/lib/clickhouse/data/test3/t_order

# 数据存放目录(/var/lib/clickhouse/data/test3/t_order)
root@9e40ca366829:/var/lib/clickhouse/data/test3/t_order# ll
total 44
drwxr-x--- 13 root       root       416 May 17 12:58 ./
drwxr-x---  3 root       root        96 May 17 12:57 ../
-rw-r-----  1 root       root       101 May 17 12:58 customer_address.bin
-rw-r-----  1 root       root        60 May 17 12:58 customer_id.bin
-rw-r-----  1 root       root        66 May 17 12:58 customer_name.bin
-rw-r-----  1 root       root        77 May 17 12:58 customer_phone.bin
-rw-r-----  1 clickhouse clickhouse 144 May 17 12:58 __marks.mrk
-rw-r-----  1 root       root        60 May 17 12:58 merchant_id.bin
-rw-r-----  1 root       root        60 May 17 12:58 order_id.bin
-rw-r-----  1 root       root        78 May 17 12:58 order_no.bin
-rw-r-----  1 root       root        54 May 17 12:58 order_status.bin
-rw-r-----  1 root       root       355 May 17 12:58 sizes.json
-rw-r-----  1 root       root        60 May 17 12:58 total_money.bin

root@9e40ca366829:/var/lib/clickhouse/metadata/test3# pwd
/var/lib/clickhouse/metadata/test3

root@9e40ca366829:/var/lib/clickhouse/metadata/test3# cat t_order.sql
ATTACH TABLE _ UUID '503c2af9-98df-46f1-850b-9b5c28cd660b'
(
    `order_id` Int32,
    `order_no` String,
    `customer_id` Int32,
    `customer_name` String,
    `customer_phone` String,
    `customer_address` String,
    `merchant_id` Int32,
    `total_money` Decimal(9, 4),
    `order_status` Enum8('S' = 1, 'E' = 2)
)
ENGINE = Log
```
