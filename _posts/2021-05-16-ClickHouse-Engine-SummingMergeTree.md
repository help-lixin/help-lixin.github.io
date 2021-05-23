---
layout: post
title: 'ClickHouse SummingMergeTree表引擎(十一)'
date: 2021-05-16
author: 李新
tags:  ClickHouse
---

### (1). SummingMergeTree介绍
> 当合并SummingMergeTree表的数据片段时,ClickHouse会把所有具有相同主键的行合并为一行,该行包含了被合并的行中具有数值数据类型的列的汇总值.   
> 如果主键的组合方式使得单个键值对应于大量的行,则可以显著的减少存储空间并加快数据查询的速度.  

### (2). 案例
```
# 1. 创建库
CREATE DATABASE test;

USE test;

 2. 创建表
CREATE TABLE IF NOT EXISTS t_order (
   merchant_id        UInt32,
   total_money        Decimal(9, 4),
   create_time        DateTime
) ENGINE = SummingMergeTree(total_money)   
  PARTITION BY merchant_id
  ORDER BY (merchant_id,create_time); 
    

# 3. 插入数据(按月统计,每个租户的销量)
INSERT INTO t_order(merchant_id,total_money,create_time) 
VALUES (7,100.50,'2021-05-10 12:12:12'),
       (7,200.50,'2021-05-11 12:12:12'),
	   (7,300.50,'2021-05-12 12:12:12'),
	   (7,400.50,'2021-05-13 12:12:12'),
	   (7,500.50,'2021-05-14 12:12:12');

INSERT INTO t_order(merchant_id,total_money,create_time) 
VALUES (8,100.50,'2020-04-15 12:12:12'),
       (8,200.50,'2020-05-15 12:12:12'),
	   (8,300.50,'2020-05-11 12:12:12');


# 4. 检索数据(根据商户id进行统计,每月的销售总额)
SELECT 
   merchant_id , 
   toYYYYMM(create_time) , 
   SUM(total_money) 
FROM t_order 
GROUP BY merchant_id , toYYYYMM(create_time);

┌─merchant_id─┬─toYYYYMM(create_time)─┬─sum(total_money)─┐
│           7 │                202105 │        1502.5000 │
│           8 │                202004 │         100.5000 │
│           8 │                202005 │         501.0000 │
└─────────────┴───────────────────────┴──────────────────┘
```
### (3). 总结
> 建议这样的操作,结合MergeTree一起使用,防止数据丢失.  