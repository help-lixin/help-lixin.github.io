---
layout: post
title: 'ClickHouse Distributed分布式表引擎(十二)'
date: 2021-05-16
author: 李新
tags:  ClickHouse
---

### (1). Distributed表引擎介绍
> 分布式引擎:可以理解为数据库中的视图,一般查询都查询分布式表.分布式表引擎会将我们的查询请求路由本地表进行查询,然后进行汇总最终返回给用户.  

### (2). 案例

```
# **************************************************************
# 在建表时,通过:ON CLUSTER perftest_3shards_1replicas,即可实现,在多台机器上创建相同的表.
# 1. 创建库
#    注意:你有多少个机器,(创建库)这条语句就要到对应的宿主机器上执行多次.
#    否则,会抛出异常:There was an error on [clickhouse-N:9000]: Code: 81, e.displayText() = DB::Exception: Database test doesn't exist 
# **************************************************************
CREATE DATABASE test;

USE test;

# 2. 创建:ReplacingMergeTree引擎表(merchant_id+create_time+order_id组合即可去重)
CREATE TABLE IF NOT EXISTS t_order ON CLUSTER perftest_3shards_1replicas (
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


# 3. 创建分布式表(分片键:test.t_order.merchant_id)
CREATE TABLE IF NOT EXISTS v_order ON CLUSTER perftest_3shards_1replicas(
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
) ENGINE = Distributed(perftest_3shards_1replicas, test ,'t_order',merchant_id);

# *********************************************************************
#  注意:这里的INSERT操作对应的是分布式表(v_order)
# *********************************************************************
# 4. 插入数据(不建议单条数据的写入)
INSERT INTO v_order(order_id,order_no,customer_id,customer_name,customer_phone,customer_address,merchant_id,total_money,order_status,create_time) 
VALUES(1,'000000000010',1000,'张三','13799999999','广东省深圳市南山区',7,500.50,'S','2021-05-16 12:12:12');

INSERT INTO v_order(order_id,order_no,customer_id,customer_name,customer_phone,customer_address,merchant_id,total_money,order_status,create_time) 
VALUES(2,'000000000011',1001,'李四','137999999998','广东省广州市',8,600.50,'S','2020-04-01 11:11:11');

INSERT INTO v_order(order_id,order_no,customer_id,customer_name,customer_phone,customer_address,merchant_id,total_money,order_status,create_time) 
VALUES(3,'000000000012',1002,'赵六','137999999997','广东省珠海市',9,700.50,'E','2021-05-01 11:11:11');  

# 5. 建议批量的插入数据.
INSERT INTO v_order(order_id,order_no,customer_id,customer_name,customer_phone,customer_address,merchant_id,total_money,order_status,create_time) 
VALUES(5,'000000000015',1005,'王五','13799999996','广东省深圳市南山区',7,500.60,'S','2020-04-16 12:12:12'),
(6,'000000000016',1006,'小何','13799999995','广东省深圳市南山区',7,500.70,'E','2021-05-20 12:12:12'),
(7,'000000000017',1007,'小丽','13799999994','广东省深圳市南山区',7,500.80,'S','2021-05-22 12:12:12');

# 6. Query(在任意一台机器上执行查询,都可以查询到数据)
clickhouse-2 :) SELECT * FROM v_order;
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address─┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┐
│        2 │ 000000000011 │        1001 │ 李四          │ 137999999998   │ 广东省广州市     │           8 │    600.5000 │ 2020-04-01 11:11:11 │ S            │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴──────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address───┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┐
│        1 │ 000000000010 │        1000 │ 张三          │ 13799999999    │ 广东省深圳市南山区 │           7 │    500.5000 │ 2021-05-16 12:12:12 │ S            │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴────────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address───┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┐
│        5 │ 000000000015 │        1005 │ 王五          │ 13799999996    │ 广东省深圳市南山区 │           7 │    500.6000 │ 2020-04-16 12:12:12 │ S            │
│        6 │ 000000000016 │        1006 │ 小何          │ 13799999995    │ 广东省深圳市南山区 │           7 │    500.7000 │ 2021-05-20 12:12:12 │ E            │
│        7 │ 000000000017 │        1007 │ 小丽          │ 13799999994    │ 广东省深圳市南山区 │           7 │    500.8000 │ 2021-05-22 12:12:12 │ S            │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴────────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address─┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┐
│        3 │ 000000000012 │        1002 │ 赵六          │ 137999999997   │ 广东省珠海市     │           9 │    700.5000 │ 2021-05-01 11:11:11 │ E            │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴──────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┘
```
### (3). 验证磁盘上的数据目录
```
# clickhouse-1上的磁盘数据(仅保存租户9的数据)
[root@clickhouse-1 t_order]# pwd
/var/lib/clickhouse/data/test/t_order
[root@clickhouse-1 t_order]# ll
total 12
drwxr-x--- 2 clickhouse clickhouse 4096 May 29 14:41 9_9_9_0
drwxr-x--- 2 clickhouse clickhouse 4096 May 29 14:21 detached
-rw-r----- 1 clickhouse clickhouse    1 May 29 14:21 format_version.txt

# clickhouse-2上的磁盘数据(仅保存租户7的数据)
[root@clickhouse-2 t_order]# pwd
/var/lib/clickhouse/data/test/t_order
[root@clickhouse-2 t_order]# ll
drwxr-x--- 2 clickhouse clickhouse 4096 May 29 14:41 7_1_1_0
drwxr-x--- 2 clickhouse clickhouse 4096 May 29 14:41 7_2_2_0
drwxr-x--- 2 clickhouse clickhouse 4096 May 29 14:21 detached
-rw-r----- 1 clickhouse clickhouse    1 May 29 14:21 format_version.txt

# clickhouse-3上的磁盘数据(仅保存租户8的数据)
[root@clickhouse-3 t_order]# pwd
/var/lib/clickhouse/data/test/t_order
[root@clickhouse-3 t_order]# ll
total 12
drwxr-x--- 2 clickhouse clickhouse 4096 May 29 14:41 8_1_1_0
drwxr-x--- 2 clickhouse clickhouse 4096 May 29 14:21 detached
-rw-r----- 1 clickhouse clickhouse    1 May 29 14:21 format_version.txt
```
### (4). 总结
> 分布式表是一张虚表,但是,该表是可以实现数据分片的(从磁盘上就能证明),但是,每个分片的副本数要怎么做?  