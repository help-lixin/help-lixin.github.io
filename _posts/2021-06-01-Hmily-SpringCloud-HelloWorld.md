---
layout: post
title: 'Hmily Spring Cloud案例入门(二)'
date: 2021-06-01
author: 李新
tags:  Hmily
---

### (1). 前言
> 用户在电商平台进行购物时,会触发:创建订单,对用户的账户进行扣款以及扣相应的库存,其中,涉及到的微服务有:订单服务(order)/账户服务(account)/库存服务(inventory).  
> 在这里,我以[官方提供的demo为例](https://github.com/help-lixin/hmily/tree/master/hmily-demo/hmily-demo-springcloud),进行搭建.  

### (2). 操作步骤
1. 导入SQL
2. 配置(application.yaml)
3. 配置(hmily.yaml)
4. 启动Eureka
5. 启动微服务(order/account/inventory)
6. 测试

### (3). 导入SQL
```
# sql文件
https://github.com/dromara/hmily/blob/master/hmily-demo/sql/hmily-demo.sql
```
### (4). 配置(application.yml)
```
spring:
    main:
        allow-bean-definition-overriding: true
    datasource:
        driver-class-name:  com.mysql.jdbc.Driver
        url: jdbc:mysql://127.0.0.1:3306/hmily_order?useUnicode=true&characterEncoding=utf8
        username: root
        password: 123456
    application:
      name: order-service
```
### (5). 配置(hmily.yml)
```
repository:
  database:
    driverClassName: com.mysql.jdbc.Driver
    url : jdbc:mysql://127.0.0.1:3306/hmily?useUnicode=true&characterEncoding=utf8
    username: root
    password: 123456
    maxActive: 20
    minIdle: 10
    connectionTimeout: 30000
    idleTimeout: 600000
    maxLifetime: 1800000
```
### (6). 启动Eureka(略)

### (7). 启动微服务(略)

### (8). 测试
```
# 测试之前,先看数据库的表里的数据
# 1. 商品1的库存为1千万
mysql> SELECT * FROM hmily_stock.inventory;
+----+------------+-----------------+----------------+
| id | product_id | total_inventory | lock_inventory |
+----+------------+-----------------+----------------+
|  1 | 1          |        10000000 |              0 |
+----+------------+-----------------+----------------+

# 2. 用户10000的余额为:1千万
mysql> SELECT * FROM hmily_account.account;
+----+---------+----------+---------------+---------------------+-------------+
| id | user_id | balance  | freeze_amount | create_time         | update_time |
+----+---------+----------+---------------+---------------------+-------------+
|  1 | 10000   | 10000000 |             0 | 2017-09-18 14:54:22 | NULL        |
+----+---------+----------+---------------+---------------------+-------------+

# 3. 此时没有订单信息
mysql>  SELECT * FROM hmily_order.order;
Empty set (0.00 sec)


# 4. 正确情况下扣库存和扣账户余额(各扣5百万)
> curl -H "Content-Type: application/json" -X POST  'http://localhost:8090/order/orderPay?count=5000000&amount=5000000'
success

# 5. 正确情况下扣库存和扣账户余额(各扣5百万)
> curl -H "Content-Type: application/json" -X POST  'http://localhost:8090/order/orderPay?count=5000000&amount=5000000'
success

# 6. 查看库存(已经不存在有库存了)
mysql> SELECT * FROM hmily_stock.inventory;
+----+------------+-----------------+----------------+
| id | product_id | total_inventory | lock_inventory |
+----+------------+-----------------+----------------+
|  1 | 1          |               0 |              0 |
+----+------------+-----------------+----------------+

# 7. 查看账户(账户也已经没有余额了)
mysql> SELECT * FROM hmily_account.account;
+----+---------+---------+---------------+---------------------+---------------------+
| id | user_id | balance | freeze_amount | create_time         | update_time         |
+----+---------+---------+---------------+---------------------+---------------------+
|  1 | 10000   |       0 |             0 | 2017-09-18 14:54:22 | 2021-06-01 20:40:23 |
+----+---------+---------+---------------+---------------------+---------------------+

# 8. 查看订单(产生了两个订单)
mysql> SELECT * FROM hmily_order.order;
+----+---------------------+---------------------+--------+------------+--------------+---------+---------+
| id | create_time         | number              | status | product_id | total_amount | count   | user_id |
+----+---------------------+---------------------+--------+------------+--------------+---------+---------+
|  1 | 2021-06-01 20:40:18 | 7897147848017715200 |      4 | 1          |      5000000 | 5000000 | 10000   |
|  2 | 2021-06-01 20:40:24 | 7897148598563250176 |      4 | 1          |      5000000 | 5000000 | 10000   |
+----+---------------------+---------------------+--------+------------+--------------+---------+---------+

# *****************************************************************
# 9. 此时,数据库里余额和库存为零了,我们再发一次调用看看情况
# *****************************************************************
> curl -H "Content-Type: application/json" -X POST  'http://localhost:8090/order/orderPay?count=1&amount=1'
{"timestamp":"2021-06-01T13:07:24.477+0000","status":500,"error":"Internal Server Error","message":"余额不足！","path":"/order/orderPay"}

# 10. 库存没有出现负数
mysql> SELECT * FROM hmily_stock.inventory;
+----+------------+-----------------+----------------+
| id | product_id | total_inventory | lock_inventory |
+----+------------+-----------------+----------------+
|  1 | 1          |               0 |              0 |
+----+------------+-----------------+----------------+

# 11. 账户也没有出现负数
mysql> SELECT * FROM hmily_account.account;
+----+---------+---------+---------------+---------------------+---------------------+
| id | user_id | balance | freeze_amount | create_time         | update_time         |
+----+---------+---------+---------------+---------------------+---------------------+
|  1 | 10000   |       0 |             0 | 2017-09-18 14:54:22 | 2021-06-01 20:40:23 |
+----+---------+---------+---------------+---------------------+---------------------+

# *******************************************************************
# 12. 订单是创建成功了,但是,状态是:3
# *******************************************************************
mysql> SELECT * FROM hmily_order.order;
+----+---------------------+---------------------+--------+------------+--------------+---------+---------+
| id | create_time         | number              | status | product_id | total_amount | count   | user_id |
+----+---------------------+---------------------+--------+------------+--------------+---------+---------+
|  1 | 2021-06-01 21:07:03 | 7897363185145122816 |      4 | 1          |      5000000 | 5000000 | 10000   |
|  2 | 2021-06-01 21:07:08 | 7897363887238057984 |      4 | 1          |      5000000 | 5000000 | 10000   |
|  3 | 2021-06-01 21:07:24 | 7897366115118125056 |      3 | 1          |            1 |       1 | 10000   |
+----+---------------------+---------------------+--------+------------+--------------+---------+---------+
```

### (9). 总结
> 官网的Demo被人动了手脚,我这边有把代码给放开,才让整个流程有异常的出现(这也能看出:开源的管理能力,在打包时,强制要求所有的测试用例都要过关,才能打包成功).  