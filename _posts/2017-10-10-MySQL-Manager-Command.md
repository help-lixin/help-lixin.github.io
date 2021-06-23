---
layout: post
title: 'MySQL 管理命令mysqladmin mysqlshow的使用'
date: 2017-10-10
author: 李新
tags: MySQL
---

### (1). 概述
在这一小节,主要是学习mysqladmin的使用法.

### (2). mysqadmin命令格式
```
 mysqladmin [OPTIONS] command command....
 
# [OPTIONS]常用选项
--character-sets-dir=utf8    #  指定字符集
-c :                         # 自动运行统计次数(与-i配合使用)
-i :                         # 间隔多长时间重复执行

-h :                         #  要连接mysql的主机地址
-u :                         #  用户名
-p :                         # 密码
-P :                         # 端口

Where command is a one or more of: (Commands may be shortened)
  create databasename	Create a new database                         # 创建数据库
  drop databasename	Delete a database and all its tables              # 删除数据库
  extended-status       Gives an extended status message from the server
  flush-hosts           Flush all cached hosts
  flush-logs            Flush all logs                                 # 刷新mysql日志
  flush-status		Clear status variables
  flush-tables          Flush all tables
  flush-threads         Flush the thread cache
  flush-privileges      Reload grant tables (same as reload)
  kill id,id,...	Kill mysql threads                                # kill线程
  password [new-password] Change old password to new-password in current format   # 修改密码
  ping			Check if mysqld is alive                              # 检测状态
  processlist		Show list of active threads in server             # 显示服务器上的所有线程
  reload		Reload grant tables                                   # 重新加载权限
  refresh		Flush all tables and close and open logfiles
  shutdown		Take server down                                      # 关闭mysql
  status		Gives a short status message from the server          # 查看mysql状态
  start-slave		Start slave
  stop-slave		Stop slave
  variables             Prints variables available
  version		Get version info from server
```
### (3) mysqladmin常用命令
```
# 1. 创建测试数据库(hello-world)
lixin-macbook:~ lixin$ mysqladmin -h 127.0.0.1 -u root -p create hello-world

# 2. ping mysql
lixin-macbook:~ lixin$ mysqladmin -h 127.0.0.1 -u root -p123456 ping
	mysqld is alive

# 3. 查看mysql的版本
lixin-macbook:~ lixin$ mysqladmin -h 127.0.0.1 -u root -p123456 version
	mysqladmin  Ver 8.42 Distrib 5.7.28, for macos10.14 on x86_64
	Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.
	Oracle is a registered trademark of Oracle Corporation and/or its
	affiliates. Other names may be trademarks of their respective
	owners.
	Server version		5.7.28-log
	Protocol version	10
	Connection		127.0.0.1 via TCP/IP
	TCP port		3306
	Uptime:			58 min 11 sec
	Threads: 2  Questions: 31  Slow queries: 0  Opens: 112  Flush tables: 1  Open tables: 6  Queries per second avg: 0.008

# 4. 查看mysql status
lixin-macbook:~ lixin$ mysqladmin -h 127.0.0.1 -u root -p123456 status
	Uptime: 3498  Threads: 2  Questions: 33  Slow queries: 0  Opens: 112  Flush tables: 1  Open tables: 6  Queries per second avg: 0.009

# 5. 修改账户对应的密码(我改完后,又再改回来了)
lixin-macbook:~ lixin$ mysqladmin -h 127.0.0.1 -u root -p123456 password 111111
	Warning: Since password will be sent to server in plain text, use ssl connection to ensure password safety.

# 6. 查看任务列表
# 只获得进程ID这一列
# lixin-macbook:sysbench lixin$ mysqladmin -h 127.0.0.1 -u root -p123456 processlist |awk '/[0-9]/{print $2}'
lixin-macbook:~ lixin$ mysqladmin -h 127.0.0.1 -u root -p123456 processlist
	+----+------+-----------------+----+---------+------+----------+------------------+
	| Id | User | Host            | db | Command | Time | State    | Info             |
	+----+------+-----------------+----+---------+------+----------+------------------+
	| 20 | root | localhost       |    | Sleep   | 127  |          |                  |
	| 22 | root | localhost:50545 |    | Query   | 0    | starting | show processlist |
	+----+------+-----------------+----+---------+------+----------+------------------+

# 7. kill某个线程(可以kill多个线程)
lixin-macbook:~ lixin$ mysqladmin -h 127.0.0.1 -u root -p123456 kill 20,22
```

### (4). mysqlshow使用
```
# mysqlshow [OPTIONS] [database [table [column]]]  
# [OPTIONS]
# --count : 统计表数据行
# -k      : 显示数据库的索引 
# -t      : 显示数据库类型
# -i      : 显示数据表的额外信息


# 1. 统计employees库下所有表的rows数量
lixin-macbook:sysbench lixin$ mysqlshow -h 127.0.0.1 -u root -p123456 --count  employees
	Database: employees
	+----------------------+----------+------------+
	|        Tables        | Columns  | Total Rows |
	+----------------------+----------+------------+
	| current_dept_emp     |        4 |     300024 |
	| departments          |        2 |         11 |
	| dept_emp             |        4 |     331603 |
	| dept_emp_latest_date |        3 |     300024 |
	| dept_manager         |        4 |         24 |
	| employees            |        6 |     300024 |
	| salaries             |        4 |    2844047 |
	| titles               |        4 |     443308 |
	+----------------------+----------+------------+

# 2. 统计employees库下employees表的rows
lixin-macbook:sysbench lixin$ mysqlshow -h 127.0.0.1 -u root -p123456 --count  employees employees
	# *************************************************************
	Database: employees  Table: employees  Rows: 300024
	# *************************************************************
	+------------+---------------+-----------------+------+-----+---------+-------+---------------------------------+---------+
	| Field      | Type          | Collation       | Null | Key | Default | Extra | Privileges                      | Comment |
	+------------+---------------+-----------------+------+-----+---------+-------+---------------------------------+---------+
	| emp_no     | int(11)       |                 | NO   | PRI |         |       | select,insert,update,references |         |
	| birth_date | date          |                 | NO   |     |         |       | select,insert,update,references |         |
	| first_name | varchar(14)   | utf8_general_ci | NO   |     |         |       | select,insert,update,references |         |
	| last_name  | varchar(16)   | utf8_general_ci | NO   |     |         |       | select,insert,update,references |         |
	| gender     | enum('M','F') | utf8_general_ci | NO   |     |         |       | select,insert,update,references |         |
	| hire_date  | date          |                 | NO   |     |         |       | select,insert,update,references |         |
	+------------+---------------+-----------------+------+-----+---------+-------+---------------------------------+---------+

# 3. 显示:employees库下employees表的索引信息
lixin-macbook:sysbench lixin$ mysqlshow -h 127.0.0.1 -u root -p123456  -k  employees employees
	Database: employees  Table: employees
	+------------+---------------+-----------------+------+-----+---------+-------+---------------------------------+---------+
	| Field      | Type          | Collation       | Null | Key | Default | Extra | Privileges                      | Comment |
	+------------+---------------+-----------------+------+-----+---------+-------+---------------------------------+---------+
	| emp_no     | int(11)       |                 | NO   | PRI |         |       | select,insert,update,references |         |
	| birth_date | date          |                 | NO   |     |         |       | select,insert,update,references |         |
	| first_name | varchar(14)   | utf8_general_ci | NO   |     |         |       | select,insert,update,references |         |
	| last_name  | varchar(16)   | utf8_general_ci | NO   |     |         |       | select,insert,update,references |         |
	| gender     | enum('M','F') | utf8_general_ci | NO   |     |         |       | select,insert,update,references |         |
	| hire_date  | date          |                 | NO   |     |         |       | select,insert,update,references |         |
	+------------+---------------+-----------------+------+-----+---------+-------+---------------------------------+---------+
	+-----------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
	| Table     | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
	+-----------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
	| employees | 0          | PRIMARY  | 1            | emp_no      | A         | 299025      |          |        |      | BTREE      |         |               |
	+-----------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+

# 4. 显示表的额外信息
lixin-macbook:sysbench lixin$ mysqlshow -h 127.0.0.1 -u root -p123456  -i  employees employees
	Database: employees  Wildcard: employees
	+-----------+--------+---------+------------+--------+----------------+-------------+-----------------+--------------+-----------+----------------+---------------------+-------------+------------+-----------------+----------+----------------+---------+
	| Name      | Engine | Version | Row_format |     | Avg_row_length | Data_length | Max_data_length | Index_length | Data_free | Auto_increment | Create_time         | Update_time | Check_time | Collation       | Checksum | Create_options | Comment |
	+-----------+--------+---------+------------+--------+----------------+-------------+-----------------+--------------+-----------+----------------+---------------------+-------------+------------+-----------------+----------+----------------+---------+
	| employees | InnoDB | 10      | Dynamic    | 299025 | 50             | 15220736    | 0               | 0            | 4194304   |                | 2021-06-15 20:03:14 |             |            | utf8_general_ci |          |                |         |
	+-----------+--------+---------+------------+--------+----------------+-------------+-----------------+--------------+-----------+----------------+---------------------+-------------+------------+-----------------+----------+----------------+---------+
```