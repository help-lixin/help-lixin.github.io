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
> <font color='red'>MergeTree为什么不建议单行的插入?因为:每一条sql语句都会生成一个分区目录,以及存储数据,在后续(8分钟左右),会对分区进行合并.</font>   

```
# 1. 插入数据(不建议单条数据的写入)
INSERT INTO t_order(order_id,order_no,customer_id,customer_name,customer_phone,customer_address,merchant_id,total_money,order_status,create_time) 
VALUES(1,'000000000010',1000,'张三','13799999999','广东省深圳市南山区',7,500.50,'S','2021-05-16 12:12:12');

INSERT INTO t_order(order_id,order_no,customer_id,customer_name,customer_phone,customer_address,merchant_id,total_money,order_status,create_time) 
VALUES(2,'000000000011',1001,'李四','137999999998','广东省广州市',8,600.50,'S','2020-04-01 11:11:11');

INSERT INTO t_order(order_id,order_no,customer_id,customer_name,customer_phone,customer_address,merchant_id,total_money,order_status,create_time) 
VALUES(3,'000000000012',1002,'赵六','137999999997','广东省珠海市',9,700.50,'E','2021-05-01 11:11:11');  

# 2. 建议批量的插入数据.
-- INSERT INTO t_order(order_id,order_no,customer_id,customer_name,customer_phone,customer_address,merchant_id,total_money,order_status,create_time) 
-- VALUES(5,'000000000015',1005,'王五','13799999996','广东省深圳市南山区',7,500.60,'S','2020-04-16 12:12:12'),
-- (6,'000000000016',1006,'小何','13799999995','广东省深圳市南山区',7,500.70,'E','2021-05-20 12:12:12'),
-- (7,'000000000017',1007,'小丽','13799999994','广东省深圳市南山区',7,500.80,'S','2021-05-22 12:12:12');
```
### (3). 查看数据目录,并验证
```
# 4. 数据库(test4.t_order)
root@9e40ca366829:/var/lib/clickhouse/data/test4/t_order# pwd
/var/lib/clickhouse/data/test4/t_order

# 5. 查看t_order对应的所有目录(会对时间进行分区,存放数据),数据被分成三个区(202004/202105/202105)
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


# 7. 优化表(手动对表进行优化压缩处理)
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


# 9. 查看数据目录()
root@9e40ca366829:/var/lib/clickhouse/data/test4/t_order# ll
drwxr-x--- 11 root       root       352 May 19 08:11 202004_2_2_0/
drwxr-x--- 11 root       root       352 May 19 08:10 202105_1_1_0/
drwxr-x--- 11 root       root       352 May 19 08:20 202105_1_3_1/
drwxr-x--- 11 root       root       352 May 19 08:11 202105_3_3_0/
drwxr-x---  2 clickhouse clickhouse  64 May 19 08:10 detached/

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
# 1. 缷载分区之前查看数据
9e40ca366829 :) SELECT * FROM t_order;
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address─┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┐
│        2 │ 000000000011 │        1001 │               │ 137999999998   │ 广东省广州市     │           8 │    600.5000 │ 2020-04-01 11:11:11 │ S            │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴──────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address───┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┐
│        1 │ 000000000010 │        1000 │ 张三          │ 13799999999    │ 广东省深圳市南山区 │           7 │    500.5000 │ 2021-05-16 12:12:12 │ S            │
│        3 │ 000000000012 │        1002 │ 赵六          │ 137999999997   │ 广东省珠海市       │           9 │    700.5000 │ 2021-05-01 11:11:11 │ E            │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴────────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┘

# 2. 缷载分区
9e40ca366829 :) ALTER TABLE t_order DETACH PARTITION '202004';
Ok.

# 3. 缷载分区后,验证结果
9e40ca366829 :) SELECT * FROM t_order;
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address───┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┐
│        1 │ 000000000010 │        1000 │ 张三          │ 13799999999    │ 广东省深圳市南山区 │           7 │    500.5000 │ 2021-05-16 12:12:12 │ S            │
│        3 │ 000000000012 │        1002 │ 赵六          │ 137999999997   │ 广东省珠海市       │           9 │    700.5000 │ 2021-05-01 11:11:11 │ E            │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴────────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┘

# 4. 缷载分区,验证文件目录
root@9e40ca366829:/var/lib/clickhouse/data/test4/t_order# pwd
/var/lib/clickhouse/data/test4/t_order

# 缷载分区后的数据目录,被移动到了:detached目录下
root@9e40ca366829:/var/lib/clickhouse/data/test4/t_order# ll detached/
total 0
drwxr-x---  3 root       root        96 May 18 08:37 ./
drwxr-x---  6 clickhouse clickhouse 192 May 18 08:37 ../
drwxr-x--- 11 root       root       352 May 18 08:37 202004_4_4_0_5/
```

### (8). 重新加载(缷载过的分区)分区
```
# 1. 查看,重新加载分区数据之前的内容.
9e40ca366829 :) SELECT * FROM t_order;
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address───┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┐
│        1 │ 000000000010 │        1000 │ 张三          │ 13799999999    │ 广东省深圳市南山区 │           7 │    500.5000 │ 2021-05-16 12:12:12 │ S            │
│        3 │ 000000000012 │        1002 │ 赵六          │ 137999999997   │ 广东省珠海市       │           9 │    700.5000 │ 2021-05-01 11:11:11 │ E            │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴────────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┘

# 2. 重新加载,被缷载过的分区
9e40ca366829 :) ALTER TABLE t_order ATTACH PARTITION '202004';
Ok.

# 3. 验证数据
9e40ca366829 :) SELECT * FROM t_order;
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address─┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┐
│        2 │ 000000000011 │        1001 │               │ 137999999998   │ 广东省广州市     │           8 │    600.5000 │ 2020-04-01 11:11:11 │ S            │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴──────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address───┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┐
│        1 │ 000000000010 │        1000 │ 张三          │ 13799999999    │ 广东省深圳市南山区 │           7 │    500.5000 │ 2021-05-16 12:12:12 │ S            │
│        3 │ 000000000012 │        1002 │ 赵六          │ 137999999997   │ 广东省珠海市       │           9 │    700.5000 │ 2021-05-01 11:11:11 │ E            │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴────────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┘
```
### (9). 删除分区
```
# 1. 删除分区之前,先查看下数据
9e40ca366829 :) SELECT * FROM t_order;
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address───┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┐
│        1 │ 000000000010 │        1000 │ 张三          │ 13799999999    │ 广东省深圳市南山区 │           7 │    500.5000 │ 2021-05-16 12:12:12 │ S            │
│        3 │ 000000000012 │        1002 │ 赵六          │ 137999999997   │ 广东省珠海市       │           9 │    700.5000 │ 2021-05-01 11:11:11 │ E            │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴────────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address─┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┐
│        2 │ 000000000011 │        1001 │ 李四          │ 137999999998   │ 广东省广州市     │           8 │    600.5000 │ 2020-04-01 11:11:11 │ S            │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴──────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┘

# 2. 删除分区
9e40ca366829 :) ALTER TABLE t_order DROP PARTITION '202004';
Ok.

# 3. 检查数据(分区被删除,数据也已经不存在了)
9e40ca366829 :) SELECT * FROM t_order;
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address───┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┐
│        1 │ 000000000010 │        1000 │ 张三          │ 13799999999    │ 广东省深圳市南山区 │           7 │    500.5000 │ 2021-05-16 12:12:12 │ S            │
│        3 │ 000000000012 │        1002 │ 赵六          │ 137999999997   │ 广东省珠海市       │           9 │    700.5000 │ 2021-05-01 11:11:11 │ E            │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴────────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┘
```

### (10). mutation(update/delete)操作
> 仅MergeTree引擎支持:mutation操作.   
> 注意:不太建议进行mutaion操作,因为这个操作过程是异步的,并且,还是非常耗时的.

```
# 1. 操作之前查看数据
9e40ca366829 :) SELECT * FROM t_order_2020_04;
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address─┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┐
│        2 │ 000000000011 │        1001 │ 李四          │ 137999999998   │ 广东省广州市     │           8 │    600.5000 │ 2020-04-01 11:11:11 │ S            │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴──────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address─┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┐
│        2 │ 000000000011 │        1001 │               │ 137999999998   │ 广东省广州市     │           8 │    600.5000 │ 2020-04-01 11:11:11 │ S            │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴──────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┘
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address───┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┐
│        1 │ 000000000010 │        1000 │ 张三          │ 13799999999    │ 广东省深圳市南山区 │           7 │    500.5000 │ 2021-05-16 12:12:12 │ S            │
│        3 │ 000000000012 │        1002 │ 赵六          │ 137999999997   │ 广东省珠海市       │           9 │    700.5000 │ 2021-05-01 11:11:11 │ E            │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴────────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┘

# 2. 删除操作
9e40ca366829 :) ALTER TABLE t_order_2020_04 DELETE WHERE order_id = 2;
Ok.

# 3. 验证数据结果(order_id为2的数据被删除了)
9e40ca366829 :) SELECT * FROM t_order_2020_04;
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address───┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┐
│        1 │ 000000000010 │        1000 │ 张三          │ 13799999999    │ 广东省深圳市南山区 │           7 │    500.5000 │ 2021-05-16 12:12:12 │ S            │
│        3 │ 000000000012 │        1002 │ 赵六          │ 137999999997   │ 广东省珠海市       │           9 │    700.5000 │ 2021-05-01 11:11:11 │ E            │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴────────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┘

# 4. 修改操作(不允许修改:主键/分区字段)
9e40ca366829 :) ALTER TABLE t_order_2020_04 UPDATE order_no='0000000000122'  WHERE order_id = 3;
Ok.

# 5. 验证数据结果(order_id为3的order_no被改成:0000000000122)
9e40ca366829 :) SELECT * FROM t_order_2020_04;
┌─order_id─┬─order_no──────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address───┬─merchant_id─┬─total_money─┬─────────create_time─┬─order_status─┐
│        1 │ 000000000010  │        1000 │ 张三          │ 13799999999    │ 广东省深圳市南山区 │           7 │    500.5000 │ 2021-05-16 12:12:12 │ S            │
│        3 │ 0000000000122 │        1002 │ 赵六          │ 137999999997   │ 广东省珠海市       │           9 │    700.5000 │ 2021-05-01 11:11:11 │ E            │
└──────────┴───────────────┴─────────────┴───────────────┴────────────────┴────────────────────┴─────────────┴─────────────┴─────────────────────┴──────────────┘

# 6. mutation原理
# 6.1 未进行mutation操作之前
root@9e40ca366829:/var/lib/clickhouse/data/test4/t_order_2020_04# ll
drwxr-x--- 11 root       root       352 May 22 04:32 202004_3_3_0/
drwxr-x---  2 clickhouse clickhouse  64 May 22 04:32 detached/
-rw-r-----  1 root       root         1 May 22 04:32 format_version.txt

# 6.2 未进行mutation操作之后
root@9e40ca366829:/var/lib/clickhouse/data/test4/t_order_2020_04# ll
drwxr-x--- 11 root       root       352 May 22 04:32 202004_2_2_0/
drwxr-x--- 11 root       root       352 May 22 04:35 202004_2_2_0_4/

drwxr-x--- 11 root       root       352 May 22 04:32 202004_3_3_0/
drwxr-x--- 11 root       root       352 May 22 04:35 202004_3_3_0_4/

drwxr-x---  2 clickhouse clickhouse  64 May 22 04:32 detached/
-rw-r-----  1 root       root         1 May 22 04:32 format_version.txt
-rw-r-----  1 root       root       115 May 22 04:35 mutation_4.txt


# 6.3  注意:
#   mutation_4.txt                 : 为SQL脚本
#   202004_2_2_0 / 202004_2_2_0_4  : 为数据目录
root@9e40ca366829:/var/lib/clickhouse/data/test4/t_order_2020_04# cat mutation_4.txt
format version: 1
create time: 2021-05-22 04:35:35
commands: UPDATE order_no = \'0000000000122\' WHERE order_id = 3

# 7. 原理
#  当执行mutation操作时,会产生一个执行计划,在数据表的根目录下:会以 mutation_id为名生成与之对应的日志文件用于记录.
#  而数据删除的过程是以数据表的每个分区目录为单位,将所有目录重写为新的目录.新目录的命名规:是在原有名称上加上system.mutations.block_number.number
#  数据在重写的过程中会将需要删除的数据去掉,旧的数据目录并不会立即删除,而是会被标记成非激活状态(active等到 MergeTree引擎的下一次合并动作触发时,这些非激活目录才会被真正从物理意义上删除. 

9e40ca366829 :) SELECT database,table,mutation_id,block_numbers.number,is_done as num FROM system.mutations;
┌─database─┬─table───────────┬─mutation_id────┬─block_numbers.number─┬─num─┐
│ test4    │ t_order_2020_04 │ mutation_4.txt │ [4]                  │   1 │
└──────────┴─────────────────┴────────────────┴──────────────────────┴─────┘

```
### (11). ClickHouse分区目录详解
> .mrk3 对 .bin   进行了索引.     
> .idx  对 .mrk3  进行了索引.   

```
# data.bin                 : 存储所有的数据文件.   
# data.mrk3                : [MarkRange]标记对应的*.bin文件中数据(块)的位置(start,end).   
# primary.idx              : 主键索引(跳跃表).

# checksums.txt            : 校验文件,保证校验数据的正确性和完整性.
# columns.txt              : 记录当前表的所有字段.
# count.txt                : 记录当前分区的行数.  
# partition.dat            : 分区信息
# minmax_create_time.idx   : 记录当前数据的分区范围.

root@9e40ca366829:/var/lib/clickhouse/data/test4/t_order/202105_1_5_1# ll
-rw-r-----  1 root       root       260 May 19 08:33 checksums.txt
-rw-r-----  1 root       root       276 May 19 08:33 columns.txt
-rw-r-----  1 root       root         1 May 19 08:33 count.txt
-rw-r-----  1 root       root       481 May 19 08:33 data.bin
-rw-r-----  1 root       root       336 May 19 08:33 data.mrk3
-rw-r-----  1 root       root        10 May 19 08:33 default_compression_codec.txt
-rw-r-----  1 root       root         8 May 19 08:33 minmax_create_time.idx
-rw-r-----  1 root       root         4 May 19 08:33 partition.dat
-rw-r-----  1 root       root         8 May 19 08:33 primary.idx
```
### (12). 分区目录命名详解
```
# PartitionID_MinBlockNum_MaxBlockNum_Level
# PartitionID: 分区ID
# MinBlockNum: 同一分区中最小数据区编号(全局计数器)
# MaxBlockNum: 同一分区中最大数据块编号(全局计数器)
# Level: 合并的次数(以分区为计数器)
root@9e40ca366829:/var/lib/clickhouse/data/test4/t_order# ll
drwxr-x--- 11 root       root       352 May 22 04:32 202004_4_4_0/
drwxr-x--- 11 root       root       352 May 22 05:12 202105_1_5_1/

# 演示分区名称的变化.
CREATE DATABASE partition_v5;

USE partition_v5;


CREATE TABLE partition_v5 ( 
ID String,
URL String,
EventTime Date
) ENGINE = MergeTree() 
PARTITION BY toYYYYMM(EventTime)
ORDER BY ID;

# 1. 刚创建库时,目录样纸.
root@9e40ca366829:/var/lib/clickhouse/data/partition_v5/partition_v5# ll

# 2. 增加一条数据(2019-05-01)
# INSERT INTO partition_v5 VALUES('A','C1','2019-05-01');
root@9e40ca366829:/var/lib/clickhouse/data/partition_v5/partition_v5# ll
drwxr-x--- 11 root       root       352 May 22 05:45 201905_1_1_0/

# 3. 增加一条数据(2019-05-02)
# INSERT INTO partition_v5 VALUES('B','C1','2019-05-02');
root@9e40ca366829:/var/lib/clickhouse/data/partition_v5/partition_v5# ll
drwxr-x--- 11 root       root       352 May 22 05:45 201905_1_1_0/
drwxr-x--- 11 root       root       352 May 22 05:46 201905_2_2_0/

# 4. 增加一条数据(2019-06-01)
# INSERT INTO partition_v5 VALUES('C','C1','2019-06-01');
root@9e40ca366829:/var/lib/clickhouse/data/partition_v5/partition_v5# ll
drwxr-x--- 11 root       root       352 May 22 05:45 201905_1_1_0/
drwxr-x--- 11 root       root       352 May 22 05:46 201905_2_2_0/
drwxr-x--- 11 root       root       352 May 22 05:46 201906_3_3_0/

# 5. 模拟数据合并操作(201905_1_1_0和201905_2_2_0,合并成了:201905_1_2_1)
#     合并的必骤是:
#      1. 要合并的分区:201905_1_1_0 / 201905_2_2_0
#      2. 取要合并分区里,最小分区号和最大分区号: 1_2
#      3. 取要合并分区里,最大的Level,并+1
#      4. 最后结果是: 201905_1_2_1
#     已经存在的旧分区目录并不会立即被删除,而是在之后的某个时刻通过后台任务被删除(默认8分钟)
# 9e40ca366829 :) OPTIMIZE TABLE partition_v5;
root@9e40ca366829:/var/lib/clickhouse/data/partition_v5/partition_v5# ll
drwxr-x--- 11 root       root       352 May 22 05:45 201905_1_1_0/
drwxr-x--- 11 root       root       352 May 22 05:47 201905_1_2_1/
drwxr-x--- 11 root       root       352 May 22 05:46 201905_2_2_0/
drwxr-x--- 11 root       root       352 May 22 05:46 201906_3_3_0/

# 6. 再插入一条2019-05-01的数据,这时,数据进入了目录:201905_4_4_0
# INSERT INTO partition_v5 VALUES('D','C1','2019-05-01');
root@9e40ca366829:/var/lib/clickhouse/data/partition_v5/partition_v5# ll
drwxr-x--- 11 root       root       352 May 22 05:47 201905_1_2_1/
drwxr-x--- 11 root       root       352 May 22 05:46 201905_2_2_0/
drwxr-x--- 11 root       root       352 May 22 05:49 201905_4_4_0/
drwxr-x--- 11 root       root       352 May 22 05:46 201906_3_3_0/

# 7. 如果这个时间点再次合并,那么最新合并的数据为:
# 201905_1_2_1 / 201905_4_4_0   --> 201905_1_4_2
```
### (13). 总结