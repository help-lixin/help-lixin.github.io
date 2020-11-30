---
layout: post
title: 'MySQL主从同步验证,并剖析Binlog内容'
date: 2020-11-01
author: 李新
tags: MySQL Canal
---


### (1).需求
> 1. 一定要打开这个参数(log_slave_updates=true),否则salve,看不到binlog信息.  
> 2. Slave向Master发起dump请求.    
> 3. Master返回binlong给Slave.    
> 4. Slave **IO线程**把binlog存放到:relaylong中.    
> 5. Slave **SQL线程**解析relaylong,并**产生binlog.**    
> 6. <font color='color'>Slave中的binlog与Master binlog是否有共识性</font>.

### (2).机器配置和端口
> IP端口以及描述

|  IP        | 端口  | 描述  |
|  ---       | ---  | ----  |
| 172.17.0.2  | 3306 | <font color='red'>Master</font> |
| 172.17.0.3  | 3306 | Slave |

---
### (3).Master配置
> /Users/lixin/DockerWorkspace/mysql/master/conf/my.cnf

```
[mysqld]
server-id=1
log-bin=master-bin
log-bin-index=master-bin.index
log_slave_updates=true
```

### (4).Slave配置
> /Users/lixin/DockerWorkspace/mysql/slave/conf/my.cnf

```
[mysqld]
server-id = 2
relay-log-index = slave-relay-bin.index
relay-log = slave-relay-bin
log_slave_updates=true

# 开启binlog
log-bin=slave-bin
log-bin-index=slave-bin.index
```


### (5).查看各个Binlog位置
> master的binlog文件当前位置在:<font color='red'>master-bin.000008<font>.    

> slave的binlog文件当前位置在:<font color='red'>slave-bin.000004</font>.

```
mysql> FLUSH LOGS;

mysql> SHOW VARIABLES  LIKE 'server_id';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 1     |
+---------------+-------+


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
| master-bin.000008 |       154 |
+-------------------+-----------+

```

```
mysql> FLUSH LOGS;

mysql> SHOW VARIABLES  LIKE 'server_id';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 2     |
+---------------+-------+

mysql> SHOW BINARY LOGS;
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| slave-bin.000001 |       201 |
| slave-bin.000002 |       177 |
| slave-bin.000003 |       177 |
| slave-bin.000004 |       154 |
+------------------+-----------+
```

### (6). master创建库(表/插入数据)
```
mysql> SHOW VARIABLES  LIKE 'server_id';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 1     |
+---------------+-------+
1 row in set (0.00 sec)

mysql> CREATE DATABASE test2;
Query OK, 1 row affected (0.01 sec)

mysql> USE test2;
Database changed
mysql> CREATE TABLE person(id INT PRIMARY KEY,name VARCHAR(25));
Query OK, 0 rows affected (0.04 sec)

mysql> INSERT INTO person(id,name) VALUES(1,"lixin");
Query OK, 1 row affected (0.02 sec)
```
### (7). Slave查看数据
```
mysql> SHOW VARIABLES  LIKE 'server_id';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 2     |
+---------------+-------+
1 row in set (0.00 sec)

mysql> SELECT * FROM test2.person;
+----+-------+
| id | name  |
+----+-------+
|  1 | lixin |
+----+-------+
1 row in set (0.00 sec)
```

### (8).获取两台机器的binlog
```
# 定义binlog临时存放目录
lixin-macbook:~ lixin$ mkdir -p ~/Downloads/binlog

# 拷贝master binlog
lixin-macbook:~ lixin$ cp ~/DockerWorkspace/mysql/master/data/master-bin.000008  ~/Downloads/binlog/

# 拷贝slave binlog
lixin-macbook:~ lixin$ cp ~/DockerWorkspace/mysql/slave/data/slave-bin.000004  ~/Downloads/binlog/

```

### (9).master binlog的具体内容
```
# 工作目录
lixin-macbook:~ lixin$ cd ~/Downloads/binlog/
# 查看binlog内容
lixin-macbook:binlog lixin$ mysqlbinlog  --no-defaults -v  --base64-output=decode-rows   master-bin.000008 |cat 
```

> master-bin.000008日志文件 


```
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=1*/;
/*!50003 SET @OLD_COMPLETION_TYPE=@@COMPLETION_TYPE,COMPLETION_TYPE=0*/;
DELIMITER /*!*/;

# at 4
#201122 17:25:54 server id 1  end_log_pos 123 CRC32 0x5feb36bb 	Start: binlog v 4, server v 5.7.9-log created 201122 17:25:54 at startup
# Warning: this binlog is either in use or was not closed properly.
ROLLBACK/*!*/;

# at 123
#201122 17:25:54 server id 1  end_log_pos 154 CRC32 0x2944e8e7 	Previous-GTIDs
# [empty]

# at 154
#201122 17:30:39 server id 1  end_log_pos 219 CRC32 0xda1b9a32 	Anonymous_GTID	last_committed=0	sequence_number=1	rbr_only=no
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;

# at 219
#201122 17:30:39 server id 1  end_log_pos 316 CRC32 0xacc68f1e 	Query	thread_id=4	exec_time=0	error_code=0
SET TIMESTAMP=1606037439/*!*/;
SET @@session.pseudo_thread_id=4/*!*/;
SET @@session.foreign_key_checks=1, @@session.sql_auto_is_null=0, @@session.unique_checks=1, @@session.autocommit=1/*!*/;
SET @@session.sql_mode=1436549152/*!*/;
SET @@session.auto_increment_increment=1, @@session.auto_increment_offset=1/*!*/;
/*!\C latin1 *//*!*/;
SET @@session.character_set_client=8,@@session.collation_connection=8,@@session.collation_server=8/*!*/;
SET @@session.lc_time_names=0/*!*/;
SET @@session.collation_database=DEFAULT/*!*/;
CREATE DATABASE test2
/*!*/;

# at 316
#201122 17:30:49 server id 1  end_log_pos 381 CRC32 0x6a923976 	Anonymous_GTID	last_committed=1	sequence_number=2	rbr_only=no
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 381
#201122 17:30:49 server id 1  end_log_pos 513 CRC32 0x4f9a4a36 	Query	thread_id=4	exec_time=0	error_code=0
use `test2`/*!*/;
SET TIMESTAMP=1606037449/*!*/;
CREATE TABLE person(id INT PRIMARY KEY,name VARCHAR(25))
/*!*/;

# at 513
#201122 17:30:53 server id 1  end_log_pos 578 CRC32 0x28d1077b 	Anonymous_GTID	last_committed=2	sequence_number=3	rbr_only=no
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;

# at 578
#201122 17:30:53 server id 1  end_log_pos 651 CRC32 0x848c5c36 	Query	thread_id=4	exec_time=0	error_code=0
SET TIMESTAMP=1606037453/*!*/;
BEGIN
/*!*/;

# at 651
#201122 17:30:53 server id 1  end_log_pos 704 CRC32 0x1a2b2c49 	Table_map: `test2`.`person` mapped to number 108

# at 704
#201122 17:30:53 server id 1  end_log_pos 750 CRC32 0xfe922041 	Write_rows: table id 108 flags: STMT_END_F
### INSERT INTO `test2`.`person`
### SET
###   @1=1
###   @2='lixin'

# at 750
#201122 17:30:53 server id 1  end_log_pos 781 CRC32 0x4f24a1d6 	Xid = 27
COMMIT/*!*/;
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
```
### (10).slave binlog日志具体内容
```
# 工作目录
lixin-macbook:~ lixin$ cd ~/Downloads/binlog/
# 查看binlog内容
lixin-macbook:binlog lixin$ mysqlbinlog  --no-defaults -v  --base64-output=decode-rows  slave-bin.000004 |cat 
```

> slave-bin.000004日志内容  

```
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=1*/;
/*!50003 SET @OLD_COMPLETION_TYPE=@@COMPLETION_TYPE,COMPLETION_TYPE=0*/;
DELIMITER /*!*/;

# at 4
#201122 17:25:57 server id 2  end_log_pos 123 CRC32 0x99dc748c 	Start: binlog v 4, server v 5.7.9-log created 201122 17:25:57 at startup
# Warning: this binlog is either in use or was not closed properly.
ROLLBACK/*!*/;

# at 123
#201122 17:25:57 server id 2  end_log_pos 154 CRC32 0xc2e90321 	Previous-GTIDs
# [empty]

# at 154
#201122 17:30:39 server id 1  end_log_pos 219 CRC32 0x142ae26f 	Anonymous_GTID	last_committed=0	sequence_number=1	rbr_only=no
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;

# at 219
#201122 17:30:39 server id 1  end_log_pos 316 CRC32 0xacc68f1e 	Query	thread_id=4	exec_time=0	error_code=0
SET TIMESTAMP=1606037439/*!*/;
SET @@session.pseudo_thread_id=4/*!*/;
SET @@session.foreign_key_checks=1, @@session.sql_auto_is_null=0, @@session.unique_checks=1, @@session.autocommit=1/*!*/;
SET @@session.sql_mode=1436549152/*!*/;
SET @@session.auto_increment_increment=1, @@session.auto_increment_offset=1/*!*/;
/*!\C latin1 *//*!*/;
SET @@session.character_set_client=8,@@session.collation_connection=8,@@session.collation_server=8/*!*/;
SET @@session.lc_time_names=0/*!*/;
SET @@session.collation_database=DEFAULT/*!*/;
CREATE DATABASE test2
/*!*/;

# at 316
#201122 17:30:49 server id 1  end_log_pos 381 CRC32 0xa4a3412b 	Anonymous_GTID	last_committed=1	sequence_number=2	rbr_only=no
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;

# at 381
#201122 17:30:49 server id 1  end_log_pos 513 CRC32 0x4f9a4a36 	Query	thread_id=4	exec_time=0	error_code=0
use `test2`/*!*/;
SET TIMESTAMP=1606037449/*!*/;
CREATE TABLE person(id INT PRIMARY KEY,name VARCHAR(25))
/*!*/;

# at 513
#201122 17:30:53 server id 1  end_log_pos 578 CRC32 0xe6e07f26 	Anonymous_GTID	last_committed=2	sequence_number=3	rbr_only=no
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;

# at 578
#201122 17:30:53 server id 1  end_log_pos 641 CRC32 0xa88533ae 	Query	thread_id=4	exec_time=0	error_code=0
SET TIMESTAMP=1606037453/*!*/;
SET @@session.sql_mode=524288/*!*/;
BEGIN
/*!*/;

# at 641
#201122 17:30:53 server id 1  end_log_pos 694 CRC32 0x18befee2 	Table_map: `test2`.`person` mapped to number 108

# at 694
#201122 17:30:53 server id 1  end_log_pos 740 CRC32 0x964157ab 	Write_rows: table id 108 flags: STMT_END_F
### INSERT INTO `test2`.`person`
### SET
###   @1=1
###   @2='lixin'

# at 740
#201122 17:30:53 server id 1  end_log_pos 771 CRC32 0x768bc29e 	Xid = 8
COMMIT/*!*/;
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
```
### (11). 分析INSERT INTO SQL语句

```
# master-bin.000008 start

    # at 704
    #201122 17:30:53 server id 1  end_log_pos 750 CRC32 0xfe922041 	Write_rows: table id 108 flags: STMT_END_F
    ### INSERT INTO `test2`.`person`
    ### SET
    ###   @1=1
    ###   @2='lixin'

# master-bin.000008 end 

# slave-bin.000004 start

    # at 694
    #201122 17:30:53 server id 1  end_log_pos 740 CRC32 0x964157ab 	Write_rows: table id 108 flags: STMT_END_F
    ### INSERT INTO `test2`.`person`
    ### SET
    ###   @1=1
    ###   @2='lixin'

# slave-bin.000004 end
```

### (12). 分析
> <font color='red'>binlog文件名称和位置没有相同点:</font>    
    master的log_name=master-bin.000008,对应的插入语句,position=704    
    slave的log_name=master-bin.000004,对应的插入语句,position=694    
> <font color='red'>binlog日志文件的时间却是相同的:</font>   
    **停止slave,对master进行操作,重新打开slave,查看master和slave日志对应的时间是否一样.**    
    master的时间为:201122 17:30:53   
    slave的时间为: 201122 17:30:53

> <font color='red'>事件类型(Write_rows)和内容是相同的</font>   
> 总结:虽然binlog名称和position不同,但是,时间(**时间只精确到了秒**)和事件类型却是相同的.