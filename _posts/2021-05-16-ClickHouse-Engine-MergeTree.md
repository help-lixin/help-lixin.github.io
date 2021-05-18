---
layout: post
title: 'ClickHouse MergeTree表引擎(五)'
date: 2021-05-16
author: 李新
tags:  ClickHouse
---

### (1). 创建MergeTree引擎表
```
# 1. 创建库
CREATE DATABASE test4;

USE test4;

 2. 创建表
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
) ENGINE = MergeTree()   
  PARTITION BY toYYYYMM(create_time)
  ORDER BY order_id ;

```
### (2). MergeTree插入数据
```
# 3. 插入数据
INSERT INTO t_order(order_id,order_no,customer_id,customer_name,customer_phone,customer_address,merchant_id,total_money,order_status,create_time) 
VALUES(1,'000000000010',1000,'张三','13799999999','广东省深圳市南山区',7,500.50,'S','2021-05-16 12:12:12');

INSERT INTO t_order(order_id,order_no,customer_id,customer_name,customer_phone,customer_address,merchant_id,total_money,order_status,create_time) 
VALUES(2,'000000000011',1001,'李四','137999999998','广东省广州市',8,600.50,'S','2020-04-01 11:11:11');

INSERT INTO t_order(order_id,order_no,customer_id,customer_name,customer_phone,customer_address,merchant_id,total_money,order_status,create_time) 
VALUES(3,'000000000012',1002,'赵六','137999999997','广东省珠海市',9,700.50,'E','2021-05-01 11:11:11');  

```
### (3). 查看数据目录,并验证
```
# 4. 数据库(test4.t_order)
root@9e40ca366829:/var/lib/clickhouse/data/test4/t_order# pwd
/var/lib/clickhouse/data/test4/t_order

# 5. 查看t_order对应的所有目录(会对时间进行分区,存放数据)
root@9e40ca366829:/var/lib/clickhouse/data/test4/t_order# ll
drwxr-x--- 11 root       root       352 May 18 06:21 202004_2_2_0/
drwxr-x--- 11 root       root       352 May 18 06:21 202105_1_1_0/
drwxr-x--- 11 root       root       352 May 18 06:21 202105_3_3_0/
drwxr-x---  2 clickhouse clickhouse  64 May 18 06:02 detached/
-rw-r-----  1 root       root         1 May 18 06:02 format_version.txt
 
# 6. 数据分成了三个区
9e40ca366829 :) SELECT * FROM t_order;
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address─┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┐
│        2 │ 000000000011 │        1001 │ 李四          │ 137999999998   │ 广东省广州市     │           8 │    600.5000 │ 2020-04-01 11:11:11 │ S            │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴──────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address───┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┐
│        1 │ 000000000010 │        1000 │ 张三          │ 13799999999    │ 广东省深圳市南山区 │           7 │    500.5000 │ 2021-05-16 12:12:12 │ S            │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴────────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address─┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┐
│        3 │ 000000000012 │        1002 │ 赵六          │ 137999999997   │ 广东省珠海市     │           9 │    700.5000 │ 2021-05-01 11:11:11 │ E            │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴──────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┘


# 7. 优化表
9e40ca366829 :) OPTIMIZE TABLE t_order;


# 8. 这时候,把1和3的数据进行了合并
9e40ca366829 :) SELECT * FROM t_order;
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address─┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┐
│        2 │ 000000000011 │        1001 │ 李四          │ 137999999998   │ 广东省广州市     │           8 │    600.5000 │ 2020-04-01 11:11:11 │ S            │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴──────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address───┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┐
│        1 │ 000000000010 │        1000 │ 张三          │ 13799999999    │ 广东省深圳市南山区 │           7 │    500.5000 │ 2021-05-16 12:12:12 │ S            │
│        3 │ 000000000012 │        1002 │ 赵六          │ 137999999997   │ 广东省珠海市       │           9 │    700.5000 │ 2021-05-01 11:11:11 │ E            │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴────────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┘
```

### (4). 查看表的分区信息

```
9e40ca366829 :)  SELECT   database,table,path,partition FROM systm.parts WHERE database = 'test4'  AND  table='t_order' \G

Row 1:
──────
database:  test4
table:     t_order
path:      /var/lib/clickhouse/store/c80/c802bf15-1c3a-4391-b507-9a7ffea952e1/202004_2_2_0/
partition: 202004                     # 对应的分区

Row 2:
──────
database:  test4
table:     t_order
path:      /var/lib/clickhouse/store/c80/c802bf15-1c3a-4391-b507-9a7ffea952e1/202105_1_3_1/
partition: 202105                     # 对应的分区
```

### (5). 复制(加载)分区数据
```
# 1. 创建2020-04的分区表
CREATE TABLE IF NOT EXISTS t_order_2020_04 (
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
) ENGINE = MergeTree()   
  PARTITION BY toYYYYMM(create_time)
  ORDER BY order_id ;

# 2. 查看:t_order表的数据(SELECT * FROM t_order WHERE formatDateTime(create_time,'%F') = '2020-04-01')
9e40ca366829 :) SELECT * FROM t_order;
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address───┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┐
│        1 │ 000000000010 │        1000 │ 张三          │ 13799999999    │ 广东省深圳市南山区 │           7 │    500.5000 │ 2021-05-16 12:12:12 │ S            │
│        3 │ 000000000012 │        1002 │ 赵六          │ 137999999997   │ 广东省珠海市       │           9 │    700.5000 │ 2021-05-01 11:11:11 │ E            │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴────────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address─┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┐
│        2 │ 000000000011 │        1001 │ 李四          │ 137999999998   │ 广东省广州市     │           8 │    600.5000 │ 2020-04-01 11:11:11 │ S            │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴──────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┘

# 3. 复制:t_order表下的'2020-04'分区里的数据到:t_order_2020_04表里
9e40ca366829 :) ALTER TABLE t_order_2020_04 REPLACE PARTITION '202004' FROM t_order;

# 4. 检查数据
9e40ca366829 :) SELECT * FROM t_order_2020_04;
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address─┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┐
│        2 │ 000000000011 │        1001 │ 李四          │ 137999999998   │ 广东省广州市     │           8 │    600.5000 │ 2020-04-01 11:11:11 │ S            │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴──────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┘
```

### (6). 清空分区中列的值

```
# 1. 清空前检查数据
9e40ca366829 :) SELECT * FROM t_order;
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address───┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┐
│        1 │ 000000000010 │        1000 │ 张三          │ 13799999999    │ 广东省深圳市南山区 │           7 │    500.5000 │ 2021-05-16 12:12:12 │ S            │
│        3 │ 000000000012 │        1002 │ 赵六          │ 137999999997   │ 广东省珠海市       │           9 │    700.5000 │ 2021-05-01 11:11:11 │ E            │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴────────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address─┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┐
│        2 │ 000000000011 │        1001 │ 李四          │ 137999999998   │ 广东省广州市     │           8 │    600.5000 │ 2020-04-01 11:11:11 │ S            │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴──────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┘


# 2. 清空t_order表'202004'分区下,所有客户姓名(customer_name)的列
9e40ca366829 :) ALTER TABLE t_order CLEAR  COLUMN customer_name  IN PARTITION '202004';
Ok.


# 3. 验证数据
9e40ca366829 :) SELECT * FROM t_order;
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address───┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┐
│        1 │ 000000000010 │        1000 │ 张三          │ 13799999999    │ 广东省深圳市南山区 │           7 │    500.5000 │ 2021-05-16 12:12:12 │ S            │
│        3 │ 000000000012 │        1002 │ 赵六          │ 137999999997   │ 广东省珠海市       │           9 │    700.5000 │ 2021-05-01 11:11:11 │ E            │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴────────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address─┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┐
│        2 │ 000000000011 │        1001 │               │ 137999999998   │ 广东省广州市     │           8 │    600.5000 │ 2020-04-01 11:11:11 │ S            │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴──────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┘
```

### (7). 缷载分区
```
# 1. 缷载分区之前,先查看下数据
9e40ca366829 :) SELECT * FROM t_order;
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address───┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┐
│        1 │ 000000000010 │        1000 │ 张三          │ 13799999999    │ 广东省深圳市南山区 │           7 │    500.5000 │ 2021-05-16 12:12:12 │ S            │
│        3 │ 000000000012 │        1002 │ 赵六          │ 137999999997   │ 广东省珠海市       │           9 │    700.5000 │ 2021-05-01 11:11:11 │ E            │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴────────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address─┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┐
│        2 │ 000000000011 │        1001 │ 李四          │ 137999999998   │ 广东省广州市     │           8 │    600.5000 │ 2020-04-01 11:11:11 │ S            │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴──────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┘

# 2. 缷载分区
9e40ca366829 :) ALTER TABLE t_order DROP PARTITION '202004';
Ok.

# 3. 检查数据(分区被缷载,数据也已经不存在了)
9e40ca366829 :) SELECT * FROM t_order;
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address───┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┐
│        1 │ 000000000010 │        1000 │ 张三          │ 13799999999    │ 广东省深圳市南山区 │           7 │    500.5000 │ 2021-05-16 12:12:12 │ S            │
│        3 │ 000000000012 │        1002 │ 赵六          │ 137999999997   │ 广东省珠海市       │           9 │    700.5000 │ 2021-05-01 11:11:11 │ E            │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴────────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┘
```