---
layout: post
title: 'MySQL主从同步(Docker)'
date: 2020-11-01
author: 李新
tags: MySQL
---


### (1).Docker创建容器
> 基于mysql:5.7.9镜像,创建master容器 

```
docker run -p 3307:3306 --name master -v /Users/lixin/DockerWorkspace/mysql/master/conf:/etc/mysql/conf.d -v /Users/lixin/DockerWorkspace/mysql/master/logs:/var/log/mysql -v /Users/lixin/DockerWorkspace/mysql/master/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=123456 -d mysql:5.7.9
```
> 启动master容器

```
docsker start master
```

> 基于mysql:5.7.9镜像,创建slave容器,并**配置--link让两个容器可以通信**

```
docker run -p 3308:3306 --name slave  --link master  -v /Users/lixin/DockerWorkspace/mysql/slave/conf:/etc/mysql/conf.d -v /Users/lixin/DockerWorkspace/mysql/slave/logs:/var/log/mysql -v /Users/lixin/DockerWorkspace/mysql/slave/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=123456 -d mysql:5.7.9
```

> 启动slave容器

```
docsker start slave
```
### (2).机器配置和端口
> IP端口以及描述

|  IP        | 端口  | 描述  |
|  ---       | ---  | ----  |
| 172.17.0.2  | 3306 | <font color='red'>Master</font> |
| 172.17.0.3  | 3306 | Slave |

---
### (2).Master配置
> /Users/lixin/DockerWorkspace/mysql/master/conf/my.cnf

```
[mysqld]
log-bin=master-bin
log-bin-index=master-bin.index
server-id=1
```
### (3).查看Master状态
```
mysql> show master status\G;
*************************** 1. row ***************************
             File: master-bin.000001
         Position: 1545
     Binlog_Do_DB: 
 Binlog_Ignore_DB: 
Executed_Gtid_Set: 
1 row in set (0.00 sec)
```
### (4).Master创建复制账号
```
mysql> CREATE USER 'repl2'@'172.17.0.3' IDENTIFIED BY 'repl2';
Query OK, 0 rows affected (0.02 sec)

mysql> GRANT REPLICATION SLAVE ON *.* TO 'repl2'@'172.17.0.3';
Query OK, 0 rows affected, 1 warning (0.01 sec)

mysql> FLUSH PRIVILEGES;
    Query OK, 0 rows affected (0.01 sec)
```
---
### (5).Slave配置
> /Users/lixin/DockerWorkspace/mysql/slave/conf/my.cnf

```
[mysqld]
server-id = 2
relay-log-index = slave-relay-bin.index
relay-log = slave-relay-bin
```
### (6).Slave命令配置
> 在172.17.0.3的机器上,可尝试通过repl2账号登录,以测试能否登录成功(mysql -u repl2 -h 172.17.0.2 -p )    
> 同步后需要关注这两个状态:    
> Slave_IO_Running: Yes    
> Slave_SQL_Running: Yes    

```
mysql> CHANGE MASTER TO MASTER_HOST='172.17.0.2',MASTER_USER='repl2',MASTER_PASSWORD='repl2',MASTER_PORT=3306,MASTER_LOG_FILE='master-bin.000001',MASTER_LOG_POS=1545;
Query OK, 0 rows affected, 2 warnings (0.09 sec)

mysql> start slave;
Query OK, 0 rows affected (0.01 sec)

mysql> SHOW SLAVE STATUS\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.17.0.2
                  Master_User: repl2
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: master-bin.000001
          Read_Master_Log_Pos: 1545
               Relay_Log_File: slave-relay-bin.000002
                Relay_Log_Pos: 321
        Relay_Master_Log_File: master-bin.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 1545
              Relay_Log_Space: 528
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 1
                  Master_UUID: 9070e2a9-1f69-11eb-8dfb-0242ac110002
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 
                Auto_Position: 0
         Replicate_Rewrite_DB: 
                 Channel_Name: 
1 row in set (0.00 sec)

```


### (7).Master模拟错误
> 在同步之前,先在在Master创建了一个数据库,Slave并没有这个库(a)
> 把Master上的a库给删了,此时Slave库将会出现错误

```
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| a                  |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.00 sec)

mysql> drop database a;
Query OK, 0 rows affected (0.03 sec)
```
### (8).Slave同步错误
```
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.00 sec)

mysql> SHOW SLAVE STATUS\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.17.0.2
                  Master_User: repl2
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: master-bin.000001
          Read_Master_Log_Pos: 1689
               Relay_Log_File: slave-relay-bin.000002
                Relay_Log_Pos: 321
        Relay_Master_Log_File: master-bin.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: No
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 1008
                   Last_Error: Error 'Can't drop database 'a'; database doesn't exist' on query. Default database: 'a'. Query: 'drop database a'
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 1545
              Relay_Log_Space: 672
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: NULL
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 1008
               Last_SQL_Error: Error 'Can't drop database 'a'; database doesn't exist' on query. Default database: 'a'. Query: 'drop database a'
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 1
                  Master_UUID: 9070e2a9-1f69-11eb-8dfb-0242ac110002
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: 
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 201105 13:51:22
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 
                Auto_Position: 0
         Replicate_Rewrite_DB: 
                 Channel_Name: 
1 row in set (0.00 sec)

```
### (9).Slave配置忽略事务
```
mysql> STOP SLAVE;
Query OK, 0 rows affected (0.00 sec)

mysql> SET GLOBAL SQL_SLAVE_SKIP_COUNTER = 1;
Query OK, 0 rows affected (0.01 sec)

mysql> START SLAVE;
Query OK, 0 rows affected (0.00 sec)

mysql> SHOW SLAVE STATUS\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.17.0.2
                  Master_User: repl2
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: master-bin.000001
          Read_Master_Log_Pos: 1689
               Relay_Log_File: slave-relay-bin.000003
                Relay_Log_Pos: 321
        Relay_Master_Log_File: master-bin.000001
             // ************************
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
            // ************************
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 1689
              Relay_Log_Space: 839
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 1
                  Master_UUID: 9070e2a9-1f69-11eb-8dfb-0242ac110002
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 
                Auto_Position: 0
         Replicate_Rewrite_DB: 
                 Channel_Name: 
1 row in set (0.00 sec)

```