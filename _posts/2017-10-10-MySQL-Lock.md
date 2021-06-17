---
layout: post
title: 'MySQL 锁详解'
date: 2017-10-10
author: 李新
tags: MySQL
---

### (1). 前言
> ["本案例的SQL脚本来源于:https://github.com/help-lixin/test_db"](https://github.com/help-lixin/test_db)

### (2). 锁是什么?
> 锁主要是用来解决多线程之间,并发访问同一共享资源而带来的数据安全问题.虽然,锁能解决数据安全问题,但是,也会带来性能的影响.

### (3). 表锁/行锁/间隙锁
> 表锁:对整张表进行加锁,写操作互斥,读取操作正常.   
> 行锁:对行数据进行加锁,写操作互斥,读取操作正常.  
> 间隙锁:这种锁是介于行锁与表锁之间的.   

### (4). 表锁(读锁)
> 当对某个线程对表进行上读锁时,其余的线程只允许读,不允许写.

```
# 事务一:
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> lock table departments read;
Query OK, 0 rows affected (0.00 sec)
```

```
# 事务二:
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> SELECT * FROM departments;
+---------+--------------------+
| dept_no | dept_name          |
+---------+--------------------+
| d009    | Customer Service   |
| d005    | Development        |
| d002    | Finance            |
| d003    | Human Resources    |
| d001    | Marketing          |
| d004    | Production         |
| d006    | Quality Management |
| d008    | Research           |
| d007    | Sales              |
+---------+--------------------+
9 rows in set (0.00 sec)

# 尝试去写,发现会阻塞在此处
mysql> INSERT INTO departments(dept_no,dept_name) VALUES('d009','IT');
```

```
# 事务一,释放当前会话的锁,事务二会继续往下执行.
mysql> unlock tables;
Query OK, 0 rows affected (0.00 sec)
```

```
# 事务二,会在事务一,释放锁后,继续往下执行.
mysql> INSERT INTO departments(dept_no,dept_name) VALUES('d009','IT');
ERROR 1062 (23000): Duplicate entry 'd009' for key 'PRIMARY'
```
### (5). 表锁(写锁)
> 当某个线程,对表进行上写锁时,其余的线程不允许读和写.

```
# 事务一,对departments表开启写锁
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> lock table departments write;
Query OK, 0 rows affected (0.00 sec)
```


```
# 事务二,查询:departments表
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

# 事务二,一直阻塞在这里.
mysql> SELECT * FROM departments;
# 中断请求.
^C^C -- query aborted
ERROR 1317 (70100): Query execution was interrupted
# 事务二,尝试加进行写操作,也会一直阻塞在这里.
mysql> INSERT INTO departments(dept_no,dept_name) VALUES('d0010','IT');
```
### (6). 行锁
> 当某个线程给某行数据加锁之后,其它线程是写(W)操作是互斥的,而读(R)操作是允许的.

```
# 事务一对表departments加行锁
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql>
mysql> SELECT * FROM departments WHERE dept_no = 'd009' FOR UPDATE;
+---------+------------------+
| dept_no | dept_name        |
+---------+------------------+
| d009    | Customer Service |
+---------+------------------+
1 row in set (0.00 sec)

```

```
# 事务二,更新departments表的数据
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

# 一直阻塞在此处,然后,抛出异常
mysql> UPDATE departments SET dept_name = 'Customer Service 2' WHERE dept_no = 'd009';
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction

# 事务二,可以读取表中所有数据.
mysql> SELECT * FROM departments;
+---------+--------------------+
| dept_no | dept_name          |
+---------+--------------------+
| d009    | Customer Service   |
| d005    | Development        |
| d002    | Finance            |
| d003    | Human Resources    |
| d001    | Marketing          |
| d004    | Production         |
| d006    | Quality Management |
| d008    | Research           |
| d007    | Sales              |
+---------+--------------------+
```
### (7). 总结
