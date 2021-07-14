---
layout: post
title: 'MySQL主从同步(ROW模式),剖析Binlog内容(二)'
date: 2020-11-01
author: 李新
tags: MySQL Canal
---

### (1). 需求
> 在binlog_format模式为:ROW模式下测试,对slave binlog进行回退,查看是否会靠成脏数据的可能性.  

### (2). 步骤
> 1. 创建一张表,插入几条数据.    
> 2. 方便观察binlog信息,把master和slave都进行(FLUSH LOGS).    
> 3. 在master执行一条UPDATE语句(更新多条数据),不明确指定version(UPDATE t_user SET name='xinli' , version=version+1 WHERE age=25).   
> 4. 检查slave表(t_user)的数据.   
> 5. slave停止与master进行同步.   
> 6. slave重新与master同步,并指定第2步FLUSH时的binlog+position(让slave回退同步).   
> 7. 再次检查slave表(t_user)数据.   
> 8. 测试alter,查看binlog信息.  

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
> MASTER_LOG_FILE:master-bin.000015   
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
| master-bin.000015 |       154 |
+-------------------+-----------+
15 rows in set (0.03 sec)
```
### (5). slave FLUSH LOGS
> MASTER_LOG_FILE:slave-bin.000014    
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
| slave-bin.000014 |       154 |
+------------------+-----------+
14 rows in set (0.03 sec)
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
# 从信息中能看到:Master_Log_File: master-bin.000015
#             Read_Master_Log_Pos: 490
mysql> SHOW SLAVE STATUS\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.17.0.2
                  Master_User: repl2
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: master-bin.000015
          Read_Master_Log_Pos: 490
               Relay_Log_File: slave-relay-bin.000051
                Relay_Log_Pos: 609
        Relay_Master_Log_File: master-bin.000015
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
          Exec_Master_Log_Pos: 490
              Relay_Log_Space: 1031
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

# 让slave同步日志回退(490变成:154)
# 重新配置slave
mysql> CHANGE MASTER TO MASTER_HOST='172.17.0.2',MASTER_USER='repl2',MASTER_PASSWORD='repl2',MASTER_PORT=3306,MASTER_LOG_FILE='master-bin.000015',MASTER_LOG_POS=154;
Query OK, 0 rows affected, 2 warnings (0.06 sec)

# 再次检查下,信息是否修改成功.
mysql> SHOW SLAVE STATUS\G;
*************************** 1. row ***************************
               Slave_IO_State:
                  Master_Host: 172.17.0.2
                  Master_User: repl2
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: master-bin.000015
          Read_Master_Log_Pos: 154
               Relay_Log_File: slave-relay-bin.000001
                Relay_Log_Pos: 4
        Relay_Master_Log_File: master-bin.000015
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
              Master_Log_File: master-bin.000015
          Read_Master_Log_Pos: 490
               Relay_Log_File: slave-relay-bin.000002
                Relay_Log_Pos: 657
        Relay_Master_Log_File: master-bin.000015
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
          Exec_Master_Log_Pos: 490
              Relay_Log_Space: 864
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
### (9). 检查slave表的数据
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
### (10). 剖析binlog内容

```
lixin-macbook:~ lixin$ cp ~/DockerWorkspace/mysql/slave/data/slave-bin.000014 ~/Downloads/binlog/
lixin-macbook:~ lixin$ cd ~/Downloads/binlog/

lixin-macbook:binlog lixin$ mysqlbinlog  --no-defaults -v  --base64-output=decode-rows slave-bin.000014 |cat

/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=1*/;
/*!50003 SET @OLD_COMPLETION_TYPE=@@COMPLETION_TYPE,COMPLETION_TYPE=0*/;
DELIMITER /*!*/;
# at 4
#210223 14:08:26 server id 2  end_log_pos 123 CRC32 0x939f23c9 	Start: binlog v 4, server v 5.7.9-log created 210223 14:08:26
# Warning: this binlog is either in use or was not closed properly.
# at 123
#210223 14:08:26 server id 2  end_log_pos 154 CRC32 0xa019926c 	Previous-GTIDs
# [empty]
# at 154
#210223 14:10:32 server id 1  end_log_pos 219 CRC32 0x43e3c3b6 	Anonymous_GTID	last_committed=0	sequence_number=1	rbr_only=no
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 219
#210223 14:10:32 server id 1  end_log_pos 282 CRC32 0x2b7c7f4a 	Query	thread_id=3	exec_time=0	error_code=0
SET TIMESTAMP=1614060632/*!*/;
SET @@session.pseudo_thread_id=3/*!*/;
SET @@session.foreign_key_checks=1, @@session.sql_auto_is_null=0, @@session.unique_checks=1, @@session.autocommit=1/*!*/;
SET @@session.sql_mode=524288/*!*/;
SET @@session.auto_increment_increment=1, @@session.auto_increment_offset=1/*!*/;
/*!\C utf8 *//*!*/;
SET @@session.character_set_client=33,@@session.collation_connection=33,@@session.collation_server=8/*!*/;
SET @@session.lc_time_names=0/*!*/;
SET @@session.collation_database=DEFAULT/*!*/;
BEGIN
/*!*/;
# at 282
#210223 14:10:32 server id 1  end_log_pos 337 CRC32 0xe4abafd3 	Table_map: `test2`.`t_user` mapped to number 108
# at 337
#210223 14:10:32 server id 1  end_log_pos 449 CRC32 0xe01d3652 	Update_rows: table id 108 flags: STMT_END_F
### UPDATE `test2`.`t_user`
### WHERE
###   @1=1
###   @2='lixin'
###   @3=25
###   @4=1
### SET
###   @1=1
###   @2='xinli'
###   @3=25
###   @4=2
### UPDATE `test2`.`t_user`
### WHERE
###   @1=2
###   @2='lixin'
###   @3=25
###   @4=1
### SET
###   @1=2
###   @2='xinli'
###   @3=25
###   @4=2
# at 449
#210223 14:10:32 server id 1  end_log_pos 480 CRC32 0x406e171f 	Xid = 21
COMMIT/*!*/;
# at 480
#210223 14:28:27 server id 1  end_log_pos 545 CRC32 0xb94fe905 	Anonymous_GTID	last_committed=1	sequence_number=2	rbr_only=no
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 545
#210223 14:28:27 server id 1  end_log_pos 608 CRC32 0x696aba9a 	Query	thread_id=3	exec_time=0	error_code=0
SET TIMESTAMP=1614061707/*!*/;
BEGIN
/*!*/;
# at 608
#210223 14:28:27 server id 1  end_log_pos 663 CRC32 0xaeb8a570 	Table_map: `test2`.`t_user` mapped to number 108
# at 663
#210223 14:28:27 server id 1  end_log_pos 736 CRC32 0x2a9f895c 	Delete_rows: table id 108 flags: STMT_END_F
### DELETE FROM `test2`.`t_user`
### WHERE
###   @1=1
###   @2='xinli'
###   @3=25
###   @4=2
### DELETE FROM `test2`.`t_user`
### WHERE
###   @1=2
###   @2='xinli'
###   @3=25
###   @4=2
# at 736
#210223 14:28:27 server id 1  end_log_pos 767 CRC32 0xc3008d75 	Xid = 37
COMMIT/*!*/;
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
```
### (11). 提取重要SQL语句
UPDATE语句(UPDATE t_user SET name='xinli',version=version+1 WHERE age=25;)

```
### UPDATE `test2`.`t_user`
### WHERE
###   @1=1
###   @2='lixin'
###   @3=25
###   @4=1
### SET
###   @1=1
###   @2='xinli'
###   @3=25
###   @4=2
### UPDATE `test2`.`t_user`
### WHERE
###   @1=2
###   @2='lixin'
###   @3=25
###   @4=1
### SET
###   @1=2
###   @2='xinli'
###   @3=25
###   @4=2
```

DELETE语句(DELETE FROM t_user WHERE age=25;)

```
### DELETE FROM `test2`.`t_user`
### WHERE
###   @1=1
###   @2='xinli'
###   @3=25
###   @4=2
### DELETE FROM `test2`.`t_user`
### WHERE
###   @1=2
###   @2='xinli'
###   @3=25
###   @4=2
```

### (13). alter剖析

```
#1. 查看master表结构和数据
mysql> SHOW VARIABLES  LIKE 'server_id';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 1     |
+---------------+-------+

# 2. 进入库
mysql> use test2;
Database changed

# 3. 查看表结构
mysql> desc t_user;
+---------+-------------+------+-----+---------+----------------+
| Field   | Type        | Null | Key | Default | Extra          |
+---------+-------------+------+-----+---------+----------------+
| id      | int(11)     | NO   | PRI | NULL    | auto_increment |
| name    | varchar(25) | YES  |     | NULL    |                |
| age     | int(3)      | YES  |     | NULL    |                |
| version | int(11)     | YES  |     | NULL    |                |
+---------+-------------+------+-----+---------+----------------+

# 4. 修改表结构之前查看表所有信息.
mysql> select * from t_user;
+----+-------+------+---------+
| id | name  | age  | version |
+----+-------+------+---------+
|  3 | xinli |   26 |       1 |
|  4 | test  |   27 |       1 |
+----+-------+------+---------+

# 5. alter表结构
mysql> ALTER TABLE t_user ADD sex CHAR(1) NOT NULL DEFAULT '0';


# 6. 查看表中数据
mysql> SELECT * FROM t_user;
+----+-------+------+---------+-----+
| id | name  | age  | version | sex |
+----+-------+------+---------+-----+
|  3 | xinli |   26 |       1 | 0   |
|  4 | test  |   27 |       1 | 0   |
+----+-------+------+---------+-----+

# ****************************************************************************************
# 1. 查看最新的binlog内容
mysql> SHOW BINARY LOGS;
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| slave-bin.000025 |       379 |
+------------------+-----------+

# 2. copy出binlog内容
lixin-macbook:~ lixin$ cp ~/DockerWorkspace/mysql/slave/data/slave-bin.000023  ~/Downloads/binlog/
lixin-macbook:~ lixin$ cd ~/Downloads/binlog/

# ************************************************************
# 3. 解析binlog内容:
#    注意:alter语句,并不会触发全表update,在binlog里反正是没有这个内容.
# ************************************************************
lixin-macbook:binlog lixin$ mysqlbinlog  --no-defaults -v  --base64-output=decode-rows slave-bin.000025 |cat
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=1*/;
/*!50003 SET @OLD_COMPLETION_TYPE=@@COMPLETION_TYPE,COMPLETION_TYPE=0*/;
DELIMITER /*!*/;
# at 4
#210714  8:53:03 server id 2  end_log_pos 123 CRC32 0xebc15af5 	Start: binlog v 4, server v 5.7.9-log created 210714  8:53:03
# Warning: this binlog is either in use or was not closed properly.
# at 123
#210714  8:53:03 server id 2  end_log_pos 154 CRC32 0x96e5ff89 	Previous-GTIDs
# [empty]
# at 154
#210714  8:53:42 server id 1  end_log_pos 219 CRC32 0x6576c267 	Anonymous_GTID	last_committed=0	sequence_number=1	rbr_only=no
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 219
#210714  8:53:42 server id 1  end_log_pos 350 CRC32 0x4c7e081d 	Query	thread_id=3	exec_time=0	error_code=0
use `test2`/*!*/;
SET TIMESTAMP=1626224022/*!*/;
SET @@session.pseudo_thread_id=3/*!*/;
SET @@session.foreign_key_checks=1, @@session.sql_auto_is_null=0, @@session.unique_checks=1, @@session.autocommit=1/*!*/;
SET @@session.sql_mode=1436549152/*!*/;
SET @@session.auto_increment_increment=1, @@session.auto_increment_offset=1/*!*/;
/*!\C utf8 *//*!*/;
SET @@session.character_set_client=33,@@session.collation_connection=33,@@session.collation_server=8/*!*/;
SET @@session.lc_time_names=0/*!*/;
SET @@session.collation_database=DEFAULT/*!*/;
ALTER TABLE t_user ADD sex CHAR(1) NOT NULL DEFAULT '0'
/*!*/;
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
```
### (13). 总结
> 在ROW模式下,对binlog进行测试结论:  
> 1. UPDATE/DELETE虽然是批量更新了多行数据,但是在ROW模式下,会变成了多条语句,并保持在一个事务以内.  
> 2. UPDATE的WHERE和SET条件都变成了整行数据,相当于在ROW模式下MySQL把SQL语句给改写了.     
> 3. DELETE的WHERE条件也被改写成整行数据.   
> 4. 从上面的结论得出:在ROW模式下,binlog+position回退的情况下,都不会出现脏数据.   
> 5. ALTER表时,是不会触发UPDATE语句的.   
