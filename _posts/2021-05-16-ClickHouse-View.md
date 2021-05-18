---
layout: post
title: 'ClickHouse 物化视图(四)'
date: 2021-05-16
author: 李新
tags:  ClickHouse
---

### (1). 创建物化视图
```
# 1. 创建物化视图
9e40ca366829 :) CREATE MATERIALIZED VIEW m_order  ENGINE = Log() POPULATE  AS SELECT * FROM t_order;
```
### (2). 查询视图
```
# 2. 查询物化视图
9e40ca366829 :) SELECT * FROM m_order;
┌─order_id─┬─order_no─────┬─customer_id─┬─customer_name─┬─customer_phone─┬─customer_address───┬─merchant_id─┬─total_money─┬─order_status─┐
│        1 │ 000000000010 │        1000 │ 张三          │ 13799999999    │ 广东省深圳市南山区 │           7 │    500.5000 │ S            │
│        2 │ 000000000011 │        1001 │ 李四          │ 137999999998   │ 广东省广州市       │           8 │    600.5000 │ S            │
└──────────┴──────────────┴─────────────┴───────────────┴────────────────┴────────────────────┴─────────────┴─────────────┴──────────────┘
```
### (3). 查看数据目录
```
# 3. clickhouse数据目录(/var/lib/clickhouse/data)
root@9e40ca366829:/var/lib/clickhouse/data/test2# pwd
/var/lib/clickhouse/data/test2

# 对应在的数据库为:test2
root@9e40ca366829:/var/lib/clickhouse/data/test2# ll
total 0
drwxr-x--- 4 clickhouse clickhouse 128 May 18 05:36 ./
drwxr-x--- 6 clickhouse clickhouse 192 May 18 05:14 ../
lrwxr-xr-x 1 root       root        66 May 18 05:36 %2Einner_id%2E1ced8981%2D2e20%2D4212%2D96e3%2D1a593a20657e -> /var/lib/clickhouse/store/40b/40b6a056-6878-42fc-9093-fc10b4403187/
lrwxr-xr-x 1 clickhouse clickhouse  66 May 17 12:36 t_order -> /var/lib/clickhouse/store/3b7/3b795b03-d582-4da2-be8f-5f208696cfe3/

# 可以看到物化视图和普通表的数据是一模一样的,相当于对:普通表的数据同步到了物化视图
root@9e40ca366829:/var/lib/clickhouse/data/test2/%2Einner_id%2E1ced8981%2D2e20%2D4212%2D96e3%2D1a593a20657e# ll
total 44
drwxr-x--- 13 root       root       416 May 18 05:36 ./
drwxr-x---  3 root       root        96 May 18 05:36 ../
-rw-r-----  1 root       root        68 May 18 05:36 customer_address.bin
-rw-r-----  1 root       root        34 May 18 05:36 customer_id.bin
-rw-r-----  1 root       root        40 May 18 05:36 customer_name.bin
-rw-r-----  1 root       root        43 May 18 05:36 customer_phone.bin
-rw-r-----  1 clickhouse clickhouse 144 May 18 05:36 __marks.mrk
-rw-r-----  1 root       root        34 May 18 05:36 merchant_id.bin
-rw-r-----  1 root       root        34 May 18 05:36 order_id.bin
-rw-r-----  1 root       root        41 May 18 05:36 order_no.bin
-rw-r-----  1 root       root        28 May 18 05:36 order_status.bin
-rw-r-----  1 root       root       354 May 18 05:36 sizes.json
-rw-r-----  1 root       root        34 May 18 05:36 total_money.bin
```