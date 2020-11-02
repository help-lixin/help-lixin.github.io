---
layout: post
title: 'MySQL主从同步'
date: 2020-11-01
author: 李新
tags: MySQL
---

### (1).机器配置和端口
> IP端口以及描述

|  IP        | 端口  | 描述  |
|  ---       | ---  | ----  |
| 172.17.0.2  | 3307 | <font color='red'>Master</font> |
| 172.17.0.3  | 3308 | Slave |

---
### (2).Master配置(/etc/my.cnf)
```
[mysqld]
log-bin=master-bin
log-bin-index=master-bin.index
server-id=1
```
### (3).查看Master状态
```
mysql> SHOW MASTER STATUS;
+-------------------+----------+--------------+------------------+-------------------+
| File              | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+-------------------+----------+--------------+------------------+-------------------+
| master-bin.000001 |      154 |              |                  |                   |
+-------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```
### (4).Master创建复制账号
```
mysql> CREATE USER repl;
    Query OK, 0 rows affected (0.02 sec)

mysql> GRANT REPLICATION SLAVE ON *.* TO 'repl@172.17.0.3' IDENTIFIED BY 'repl';
    Query OK, 0 rows affected, 1 warning (0.01 sec)

mysql> FLUSH PRIVILEGES;
    Query OK, 0 rows affected (0.01 sec)
```
---
### (5).Slave配置(/etc/my.cnf)
```
[mysqld]
server-id = 2
relay-log-index = slave-relay-bin.index
relay-log = slave-relay-bin
```
### (6).Slave命令配置
```
mysql> CHANGE MASTER TO MASTER_HOST='172.17.0.2',MASTER_USER='repl',MASTER_PASSWORD='repl',MASTER_PORT=3306,MASTER_LOG_FILE='master-bin.000001',MASTER_LOG_POS=1513;
    Query OK, 0 rows affected, 2 warnings (0.09 sec)

mysql> start slave;
    Query OK, 0 rows affected (0.01 sec)



```

### (7).
### (8).