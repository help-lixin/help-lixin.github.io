---
layout: post
title: 'ClickHouse ReplacingMergeTree表引擎(七)'
date: 2021-05-16
author: 李新
tags:  ClickHouse
---

### (1). ReplacingMergeTree介绍
> 该引擎与MergeTree的不同之处在于<font color='red'>它会去除区内,主键相同的重复项(仅在合并的时候执行).</font>  

### (2). 案例
```
CREATE DATABASE test10;

USE test10;

# 1. 创建:ReplacingMergeTree引擎表
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
   order_status       Enum('S' = 1 , 'E' = 2)
) ENGINE = ReplacingMergeTree(create_time)   
  PARTITION BY merchant_id
  ORDER BY order_id ;
  

# 2. 插入3条数据(以租户ID进行分区)
INSERT INTO t_order(order_id,order_no,customer_id,customer_name,customer_phone,customer_address,merchant_id,total_money,order_status,create_time) 
VALUES (1,'000000000010',1000,'张三','13799999999','广东省深圳市南山区',7,500.50,'S','2021-05-16 12:12:12'),
       (2,'000000000011',1001,'李四','137999999998','广东省广州市',8,600.50,'S','2020-04-01 11:11:11'),
	   (3,'000000000012',1002,'赵六','137999999997','广东省珠海市',9,700.50,'E','2021-05-01 11:11:11');  


# 3. 插入3条数据后,目录结构如下:
root@9e40ca366829:/var/lib/clickhouse/data/test10/t_order# ll
drwxr-x--- 11 root       root       352 May 22 07:56 7_1_1_0/
drwxr-x--- 11 root       root       352 May 22 07:56 8_2_2_0/
drwxr-x--- 11 root       root       352 May 22 07:56 9_3_3_0/

# 4. 尝试再插入一条数所(分区要相同,主键要相同,才会保留最大版本)
INSERT INTO t_order(order_id,order_no,customer_id,customer_name,customer_phone,customer_address,merchant_id,total_money,order_status,create_time) 
VALUES(1,'000000000010',1000,'张三','13799999999','广东省深圳市南山区',7,500.80,'S','2021-05-16 12:12:18');


# 5. 尝试再插入一条数所(分区要相同,主键要相同,才会保留最大版本),的目录结构.
root@9e40ca366829:/var/lib/clickhouse/data/test10/t_order# ll
drwxr-x--- 11 root       root       352 May 22 07:56 7_1_1_0/
drwxr-x--- 11 root       root       352 May 22 07:59 7_4_4_0/
drwxr-x--- 11 root       root       352 May 22 07:56 8_2_2_0/
drwxr-x--- 11 root       root       352 May 22 07:56 9_3_3_0/


# 6. 优化表
9e40ca366829 :) OPTIMIZE TABLE t_order;


# 7. 查看目录(7_1_1_0/7_4_4_0) --> 7_1_4_1
root@9e40ca366829:/var/lib/clickhouse/data/test10/t_order# ll
drwxr-x--- 11 root       root       352 May 22 07:56 7_1_1_0/
drwxr-x--- 11 root       root       352 May 22 08:01 7_1_4_1/
drwxr-x--- 11 root       root       352 May 22 07:59 7_4_4_0/
drwxr-x--- 11 root       root       352 May 22 07:56 8_2_2_0/
drwxr-x--- 11 root       root       352 May 22 07:56 9_3_3_0/
```
### (3). 总结
> 通过:ReplacingMergeTree实际是可以对数据进行更新的,更新的条件是:数据在同一个区,并且,是根据主键进行更新.     
> 从这么多次的INSERT和目录对比来看,ClickHouse比较适合于批量的插入数据,而不是,单行单行的插入(因为:单行数据会产生一个目录).   