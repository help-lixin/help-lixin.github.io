---
layout: post
title: 'MySQL主从同步(STATEMENT模式),剖析Binlog内容(二)'
date: 2020-11-01
author: 李新
tags: MySQL Canal
---

### (1). 需求
> 在binlog_format模式为:STATEMENT模式下测试,对slave binlog进行回退,查看是否会靠成脏数据的可能性.  

### (2). 步骤
> 1. 创建一张表,插入几条数据.    
> 2. 方便观察binlog信息,把master和slave都进行(FLUSH LOGS).    
> 3. 在master执行一条UPDATE语句(更新多条数据),不明确指定version(UPDATE t_user SET name='xinli' , version=version+1 WHERE age=25).   
> 4. 检查slave表(t_user)的数据.   
> 5. slave停止与master进行同步.   
> 6. slave重新与master同步,并指定第2步FLUSH时的binlog+position(让slave回退同步).   
> 7. 再次检查slave表(t_user)数据.   

### (3). 准备工作
```
mysql> USE test2;
Database changed
mysql> CREATE TABLE t_user(id INT PRIMARY KEY AUTO_INCREMENT,name VARCHAR(25),age INT(3),version INT);

mysql> INSERT INTO t_user(name,age,version) VALUES('lixin',25,1);
mysql> INSERT INTO t_user(name,age,version) VALUES('lixin',25,1);
mysql> INSERT INTO t_user(name,age,version) VALUES('xinli',26,1);
mysql> INSERT INTO t_user(name,age,version) VALUES('test',27,1);

mysql> SELECT * FROM t_user;
+----+-------+------+---------+
| id | name  | age  | version |
+----+-------+------+---------+
|  1 | lixin |   25 |       1 |
|  2 | lixin |   25 |       1 |
|  3 | xinli |   26 |       1 |
|  4 | test  |   27 |       1 |
+----+-------+------+---------+
```
### (4). master FLUSH LOGS
> MASTER_LOG_FILE:master-bin.000017   
> MASTER_LOG_POS:154

```
mysql> FLUSH LOGS;
Query OK, 0 rows affected (0.03 sec)


mysql> SHOW VARIABLES  LIKE 'server_id';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 1     |
+---------------+-------+
1 row in set (0.00 sec)

mysql> SHOW BINARY LOGS;
+-------------------+-----------+
| Log_name          | File_size |
+-------------------+-----------+
| master-bin.000001 |      1712 |
| master-bin.000002 |       336 |
| master-bin.000003 |       154 |
| master-bin.000004 |       177 |
| master-bin.000005 |       202 |
| master-bin.000006 |       781 |
| master-bin.000007 |      2188 |
| master-bin.000008 |       804 |
| master-bin.000009 |       449 |
| master-bin.000010 |       177 |
| master-bin.000011 |     48380 |
| master-bin.000012 |       177 |
| master-bin.000013 |      1605 |
| master-bin.000014 |      1548 |
| master-bin.000015 |       964 |
| master-bin.000016 |      1569 |
| master-bin.000017 |       154 |
+-------------------+-----------+
```
### (5). slave FLUSH LOGS
> MASTER_LOG_FILE:slave-bin.000016    
> MASTER_LOG_POS:154

```
mysql> FLUSH LOGS;
Query OK, 0 rows affected (0.07 sec)

mysql> SHOW VARIABLES  LIKE 'server_id';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 2     |
+---------------+-------+
1 row in set (0.00 sec)

mysql> SHOW BINARY LOGS;
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| slave-bin.000001 |       201 |
| slave-bin.000002 |       177 |
| slave-bin.000003 |       177 |
| slave-bin.000004 |       794 |
| slave-bin.000005 |       439 |
| slave-bin.000006 |       177 |
| slave-bin.000007 |     46375 |
| slave-bin.000008 |       177 |
| slave-bin.000009 |       201 |
| slave-bin.000010 |       979 |
| slave-bin.000011 |       518 |
| slave-bin.000012 |       460 |
| slave-bin.000013 |      1507 |
| slave-bin.000014 |       944 |
| slave-bin.000015 |      1568 |
| slave-bin.000016 |       154 |
+------------------+-----------+
```
### (6). master执行update语句
```
mysql> UPDATE t_user SET name='xinli' , version=version+1 WHERE age=25;
Query OK, 2 rows affected (0.01 sec)
Rows matched: 2  Changed: 2  Warnings: 0
```
### (7). 检查数据
```
mysql> SELECT * FROM t_user;
+----+-------+------+---------+
| id | name  | age  | version |
+----+-------+------+---------+
|  1 | xinli |   25 |       2 |
|  2 | xinli |   25 |       2 |
|  3 | xinli |   26 |       1 |
|  4 | test  |   27 |       1 |
+----+-------+------+---------+
4 rows in set (0.01 sec)
```
### (8). 重新配置slave
```
# 查看下slave同步状态
# 从信息中能看到:Master_Log_File: master-bin.000017
#             Read_Master_Log_Pos: 470
mysql> SHOW SLAVE STATUS\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.17.0.2
                  Master_User: repl2
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: master-bin.000017
          Read_Master_Log_Pos: 470
               Relay_Log_File: slave-relay-bin.000008
                Relay_Log_Pos: 589
        Relay_Master_Log_File: master-bin.000017
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
          Exec_Master_Log_Pos: 470
              Relay_Log_Space: 1011
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


mysql> STOP SLAVE;
Query OK, 0 rows affected (0.00 sec)

# 让slave同步日志回退(470变成:154)
# 重新配置slave
mysql> CHANGE MASTER TO MASTER_HOST='172.17.0.2',MASTER_USER='repl2',MASTER_PASSWORD='repl2',MASTER_PORT=3306,MASTER_LOG_FILE='master-bin.000017',MASTER_LOG_POS=154;
Query OK, 0 rows affected, 2 warnings (0.06 sec)

# 再次检查下,信息是否修改成功.
mysql> SHOW SLAVE STATUS\G;
*************************** 1. row ***************************
               Slave_IO_State:
                  Master_Host: 172.17.0.2
                  Master_User: repl2
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: master-bin.000017
          Read_Master_Log_Pos: 154
               Relay_Log_File: slave-relay-bin.000001
                Relay_Log_Pos: 4
        Relay_Master_Log_File: master-bin.000017
             Slave_IO_Running: No
            Slave_SQL_Running: No
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 154
              Relay_Log_Space: 154
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
               Last_SQL_Errno: 0
               Last_SQL_Error:
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
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set:
            Executed_Gtid_Set:
                Auto_Position: 0
         Replicate_Rewrite_DB:
                 Channel_Name:
1 row in set (0.00 sec)


# 开启同步
mysql> start slave;
Query OK, 0 rows affected (0.01 sec)

# 查看同步信息
mysql> SHOW SLAVE STATUS\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.17.0.2
                  Master_User: repl2
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: master-bin.000017
          Read_Master_Log_Pos: 470
               Relay_Log_File: slave-relay-bin.000002
                Relay_Log_Pos: 637
        Relay_Master_Log_File: master-bin.000017
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
          Exec_Master_Log_Pos: 470
              Relay_Log_Space: 844
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
### (9). 检查表t_user的数据
> 结论已出:STATEMENT模式下是:SQL语句+参数+执行上下文,很明显slave回退后,就出现了脏数据.  

```
# 查看master数据
mysql> SELECT * FROM t_user;
+----+-------+------+---------+
| id | name  | age  | version |
+----+-------+------+---------+
|  1 | xinli |   25 |       2 |
|  2 | xinli |   25 |       2 |
|  3 | xinli |   26 |       1 |
|  4 | test  |   27 |       1 |
+----+-------+------+---------+
4 rows in set (0.00 sec)

# 查看slave数据
mysql> SELECT * FROM t_user;
+----+-------+------+---------+
| id | name  | age  | version |
+----+-------+------+---------+
|  1 | xinli |   25 |       3 |
|  2 | xinli |   25 |       3 |
|  3 | xinli |   26 |       1 |
|  4 | test  |   27 |       1 |
+----+-------+------+---------+
```
### (10). 剖析binlog内容

```
lixin-macbook:~ lixin$ cp ~/DockerWorkspace/mysql/slave/data/slave-bin.000016 ~/Downloads/binlog/
lixin-macbook:~ lixin$ cd ~/Downloads/binlog/

lixin-macbook:binlog lixin$ mysqlbinlog  --no-defaults -v  --base64-output=decode-rows slave-bin.000016 |cat

/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=1*/;
/*!50003 SET @OLD_COMPLETION_TYPE=@@COMPLETION_TYPE,COMPLETION_TYPE=0*/;
DELIMITER /*!*/;
# at 4
#210223 14:52:14 server id 2  end_log_pos 123 CRC32 0x3228e264 	Start: binlog v 4, server v 5.7.9-log created 210223 14:52:14
# Warning: this binlog is either in use or was not closed properly.
# at 123
#210223 14:52:14 server id 2  end_log_pos 154 CRC32 0x40eaf46b 	Previous-GTIDs
# [empty]
# at 154
#210223 14:52:57 server id 1  end_log_pos 219 CRC32 0x0ca54452 	Anonymous_GTID	last_committed=0	sequence_number=1	rbr_only=no
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 219
#210223 14:52:57 server id 1  end_log_pos 300 CRC32 0x32411b25 	Query	thread_id=3	exec_time=0	error_code=0
SET TIMESTAMP=1614063177/*!*/;
SET @@session.pseudo_thread_id=3/*!*/;
SET @@session.foreign_key_checks=1, @@session.sql_auto_is_null=0, @@session.unique_checks=1, @@session.autocommit=1/*!*/;
SET @@session.sql_mode=1436549152/*!*/;
SET @@session.auto_increment_increment=1, @@session.auto_increment_offset=1/*!*/;
/*!\C utf8 *//*!*/;
SET @@session.character_set_client=33,@@session.collation_connection=33,@@session.collation_server=8/*!*/;
SET @@session.lc_time_names=0/*!*/;
SET @@session.collation_database=DEFAULT/*!*/;
BEGIN
/*!*/;

# at 300
#210223 14:52:57 server id 1  end_log_pos 439 CRC32 0xdc9b1dc0 	Query	thread_id=3	exec_time=0	error_code=0
use `test2`/*!*/;
SET TIMESTAMP=1614063177/*!*/;
UPDATE t_user SET name='xinli' , version=version+1 WHERE age=25
/*!*/;

# at 439
#210223 14:52:57 server id 1  end_log_pos 470 CRC32 0xc22c2be9 	Xid = 17
COMMIT/*!*/;

# at 470
#210223 14:52:57 server id 1  end_log_pos 535 CRC32 0x8b0f12e0 	Anonymous_GTID	last_committed=1	sequence_number=2	rbr_only=no
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 535
#210223 14:52:57 server id 1  end_log_pos 616 CRC32 0x0851eb97 	Query	thread_id=3	exec_time=225	error_code=0
SET TIMESTAMP=1614063177/*!*/;
BEGIN
/*!*/;
# at 616
#210223 14:52:57 server id 1  end_log_pos 755 CRC32 0x0a1f3d16 	Query	thread_id=3	exec_time=225	error_code=0
SET TIMESTAMP=1614063177/*!*/;
UPDATE t_user SET name='xinli' , version=version+1 WHERE age=25
/*!*/;
# at 755
#210223 14:52:57 server id 1  end_log_pos 786 CRC32 0x66591281 	Xid = 24
COMMIT/*!*/;

# at 786
#210223 14:58:33 server id 1  end_log_pos 851 CRC32 0xc7933cdf 	Anonymous_GTID	last_committed=2	sequence_number=3	rbr_only=no
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 851
#210223 14:58:33 server id 1  end_log_pos 932 CRC32 0xb6540d78 	Query	thread_id=3	exec_time=0	error_code=0
SET TIMESTAMP=1614063513/*!*/;
BEGIN
/*!*/;

# at 932
#210223 14:58:33 server id 1  end_log_pos 1039 CRC32 0x0b13b25f 	Query	thread_id=3	exec_time=0	error_code=0
SET TIMESTAMP=1614063513/*!*/;
DELETE FROM t_user WHERE age=25
/*!*/;
# at 1039
#210223 14:58:33 server id 1  end_log_pos 1070 CRC32 0x15303ee5 	Xid = 31
COMMIT/*!*/;

SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
```
### (11). 提出binlog重要信息
> UPDATE语句(UPDATE t_user SET name='xinli',version=version+1 WHERE age=25;)

```
# at 300
#210223 14:52:57 server id 1  end_log_pos 439 CRC32 0xdc9b1dc0 	Query	thread_id=3	exec_time=0	error_code=0
use `test2`/*!*/;
SET TIMESTAMP=1614063177/*!*/;
UPDATE t_user SET name='xinli' , version=version+1 WHERE age=25
/*!*/;
```

> DELETE语句(DELETE FROM t_user WHERE age=25;)

```
# at 616
#210223 14:52:57 server id 1  end_log_pos 755 CRC32 0x0a1f3d16 	Query	thread_id=3	exec_time=225	error_code=0
SET TIMESTAMP=1614063177/*!*/;
UPDATE t_user SET name='xinli' , version=version+1 WHERE age=25
/*!*/;
```
### (13). 总结
> 在STATEMENT模式下,对binlog进行测试结论:  
> 1. 在STATEMENT模式下,UPDATE/DELETE在binlog中的格式是:SQL语句+参数+执行上下文信息.然后,slave执行SQL语句.    
> 2. 从上面的结论得出:在STATEMENT模式下,binlog+position回退的情况下,会出现脏数据的可能性.   
> 3. 解决方案是:version字段,不能用上下文中的,需要开发先做Query,并当成参数传递. 