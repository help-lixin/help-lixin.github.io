---
layout: post
title: 'MySQL MVCC'
date: 2017-10-10
author: 李新
tags: MySQL
---

### (1). 前言
> ["本案例的SQL脚本来源于:https://github.com/help-lixin/test_db"](https://github.com/help-lixin/test_db)

### (2). 事务特性
+ 原子性: 多条SQL语句,要么一起成功,要么一起失败.  
+ 一致性:事务在提交和rollback都要保证数据是一致性的.  
+ 持久性:事务一旦提交,就要保证数据的持久,不能丢失.   
+ 隔离性: 多个线程,在并发访问下,提供了一套隔离机制.不同的隔离级别,会有不同的效果.  

### (3). 事务隔离级别
+ 未提交读(read uncommitted)   
+ 已提交读(read committed)   
+ 可重复读(repeatable read)   
+ 串行化(serializable)   

### (4). 查看和设置事务隔离级别
```
# 查看当前事务的隔离级别
mysql> SELECT @@tx_isolation;
+-----------------+
| @@tx_isolation  |
+-----------------+
| REPEATABLE-READ |
+-----------------+

# 设置事务隔离级别
mysql> set session transaction isolation level read uncommitted;
mysql> set session transaction isolation level read committed;
mysql> set session transaction isolation level repeatable read;
mysql> set session transaction isolation level serializable;
```
### (5). read uncommitted测试
> read uncommitted:就是在多线程的情况下,一个线程能读到另一个线程还未提交(commit)的数据,造成:脏读.  

```
# 事务A
# 设置隔离级别为:读取未提交
mysql> set session transaction isolation level read uncommitted;
Query OK, 0 rows affected (0.00 sec)

# 开启事务
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

# 查询当前数据
mysql> SELECT * FROM employees.departments;
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

# 插入一条数据,另开一个事务,看下,数据是否存在
mysql> INSERT INTO employees.departments(dept_no,dept_name) VALUES('d010','IT');
Query OK, 1 row affected (0.00 sec)

# 事务rollback
mysql> rollback;
Query OK, 0 rows affected (0.00 sec)
```

```
# 事务B
# 设置事务隔离级别为:读未提交
mysql> set session transaction isolation level read uncommitted;
Query OK, 0 rows affected (0.00 sec)

# 开启事务
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

# 查询表里现有数据
mysql> SELECT * FROM employees.departments;
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

# 等待事务A,插入一条数据后(未commit),再进行Query,发现:能读到事务A插入的数据.
mysql> SELECT * FROM employees.departments;
+---------+--------------------+
| dept_no | dept_name          |
+---------+--------------------+
| d009    | Customer Service   |
| d005    | Development        |
| d002    | Finance            |
| d003    | Human Resources    |
| d010    | IT                 |
| d001    | Marketing          |
| d004    | Production         |
| d006    | Quality Management |
| d008    | Research           |
| d007    | Sales              |
+---------+--------------------+
10 rows in set (0.00 sec)

# 在事务A rollback时,数据又少了一条.
mysql> SELECT * FROM employees.departments;
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
```
### (6). read committed测试
> read committed:就是在多线程的情况下,一个线程只能读到其它线程已经提交(commit)的数据,不能读取到还未提交的数据,能有效解决脏读问题.  
> 缺陷:在同一个事务中,读取到两次不同的结果,这种现象叫:不可重复读.  

```
# 事务A
# 设置事务隔离级别
mysql> set session transaction isolation level read committed;
Query OK, 0 rows affected (0.00 sec)

# 开启事务
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

# 先看下表里的数据
mysql> SELECT * FROM employees.departments;
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

# 插入一条数据
mysql> INSERT INTO employees.departments(dept_no,dept_name) VALUES('d010','IT');
Query OK, 1 row affected (0.00 sec)

# 在自己当前事务能看到插入的数据
mysql> SELECT * FROM employees.departments;
+---------+--------------------+
| dept_no | dept_name          |
+---------+--------------------+
| d009    | Customer Service   |
| d005    | Development        |
| d002    | Finance            |
| d003    | Human Resources    |
| d010    | IT                 |
| d001    | Marketing          |
| d004    | Production         |
| d006    | Quality Management |
| d008    | Research           |
| d007    | Sales              |
+---------+--------------------+
10 rows in set (0.00 sec)

```

```
# 事务B
# 设置事务隔离级别
mysql> set session transaction isolation level read committed;
Query OK, 0 rows affected (0.00 sec)

# 开启事务
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

# 事务A已经插入了一条数据(未commit),验证事务B是否会看到最新插入的数据.
# 发现:在read committed的情况下,各个线程之间互不干扰.
mysql> SELECT * FROM employees.departments;
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
```
### (7). repeatable read测试
> repeatable read:一个线程开启事务后(未commit之前),永远都只能读取到:开启事务时,已经提交的数据.  
> 而在开启事务之后,就算,其余线程提交(commit)的数据,不论读取多少次,结果集都是相同(读不到其余线程commit的数据).  
> 缺陷:

```
# 设置事务隔离级别
# 事务A
mysql> set session transaction isolation level repeatable read;
Query OK, 0 rows affected (0.00 sec)

# 事务A开启事务
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> INSERT INTO employees.departments(dept_no,dept_name) VALUES('d011','IT-2');
Query OK, 1 row affected (0.00 sec)

mysql> commit;
Query OK, 0 rows affected (0.01 sec)
```

```
# 事务B
mysql> set session transaction isolation level repeatable read;
Query OK, 0 rows affected (0.00 sec)

mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> SELECT * FROM employees.departments;
+---------+--------------------+
| dept_no | dept_name          |
+---------+--------------------+
| d009    | Customer Service   |
| d005    | Development        |
| d002    | Finance            |
| d003    | Human Resources    |
| d010    | IT                 |
| d001    | Marketing          |
| d004    | Production         |
| d006    | Quality Management |
| d008    | Research           |
| d007    | Sales              |
+---------+--------------------+
10 rows in set (0.00 sec)

# 事务A已经commit了,我(事务B)却仍然读不到数据
mysql> SELECT * FROM employees.departments;
+---------+--------------------+
| dept_no | dept_name          |
+---------+--------------------+
| d009    | Customer Service   |
| d005    | Development        |
| d002    | Finance            |
| d003    | Human Resources    |
| d010    | IT                 |
| d001    | Marketing          |
| d004    | Production         |
| d006    | Quality Management |
| d008    | Research           |
| d007    | Sales              |
+---------+--------------------+
```
### (8). MVCC
> MySQL在读和写的操作中,对读的性能做了并发的保障,让所有的读都是快照读.而对于写的时候,进行版本控制,如果,真实数据的版本比快照版本更加的新,那么在写之前,会同步真实数据到快照版本里.  
> 这样既能提高读的并发性,又能够保证写的数据安全.   
> 其实,我觉得MVCC和Java内存模型(volatile)很像.  
### (9). 总结