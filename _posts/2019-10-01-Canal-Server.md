---
layout: post
title: 'Canal Server 启动'
date: 2019-10-01
author: 李新
tags: Canal
---

### (1). Canal原理图

!["Canal 原理图"](/assets/canal/imgs/canal.png)

### (2). MySQL Master开启binlog
```
[mysqld]
log-bin=mysql-bin
binlog_format=row
server-id=1
```

### (3). 查看(Master)binlog配置是否生效(以及常用命令)

> 查看Master状态    
> mysql> SHOW MASTER STATUS\G;

```
*************************** 1. row ***************************
             File: mysql-bin.000012
         Position: 777
     Binlog_Do_DB: 
 Binlog_Ignore_DB: 
Executed_Gtid_Set: 
1 row in set (0.00 sec)
```

> 查看是否开启了binlog    
> mysql> SHOW VARIABLES LIKE 'log_bin';

```
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| log_bin       | ON    |
+---------------+-------+
1 row in set (0.00 sec)
```

> 查看日志格式是否为:row    
> mysql> SHOW VARIABLES LIKE '%binlog_format%';

```
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| binlog_format | ROW   |
+---------------+-------+
1 row in set (0.00 sec)
```

> 查看server_id   
> mysql> SHOW VARIABLES LIKE 'server_id';

```
+----------------+-------+
| Variable_name  | Value |
+----------------+-------+
| server_id      | 1     |
+----------------+-------+
1 rows in set (0.00 sec)

```

> 查看master所有的日志文件    
> mysql> SHOW BINARY LOGS;

```
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000001 |       177 |
| mysql-bin.000002 |       177 |
| mysql-bin.000003 |       177 |
| mysql-bin.000004 |       177 |
| mysql-bin.000005 |       177 |
| mysql-bin.000006 |       177 |
| mysql-bin.000007 |       820 |
| mysql-bin.000008 |       177 |
| mysql-bin.000009 |      1954 |
| mysql-bin.000010 |       177 |
| mysql-bin.000011 |       177 |
| mysql-bin.000012 |       777 |
+------------------+-----------+
12 rows in set (0.00 sec)
```

> 查看具体某个bin log文件内容    
> mysql> SHOW BINLOG EVENTS IN 'mysql-bin.000012'\G;

```
*************************** 1. row ***************************
   Log_name: mysql-bin.000012
        Pos: 4
 Event_type: Format_desc
  Server_id: 1
End_log_pos: 123
       Info: Server ver: 5.7.28-log, Binlog ver: 4
*************************** 2. row ***************************
   Log_name: mysql-bin.000012
        Pos: 123
 Event_type: Previous_gtids
  Server_id: 1
End_log_pos: 154
       Info: 
*************************** 3. row ***************************
   Log_name: mysql-bin.000012
        Pos: 154
 Event_type: Anonymous_Gtid
  Server_id: 1
End_log_pos: 219
       Info: SET @@SESSION.GTID_NEXT= 'ANONYMOUS'
*************************** 4. row ***************************
   Log_name: mysql-bin.000012
        Pos: 219
 Event_type: Query
  Server_id: 1
End_log_pos: 400
       Info: CREATE USER 'canal'@'%' IDENTIFIED WITH 'mysql_native_password' AS '*E3619321C1A937C46A0D8BD1DAC39F93B27D4458'
*************************** 5. row ***************************
   Log_name: mysql-bin.000012
        Pos: 400
 Event_type: Anonymous_Gtid
  Server_id: 1
End_log_pos: 465
       Info: SET @@SESSION.GTID_NEXT= 'ANONYMOUS'
*************************** 6. row ***************************
   Log_name: mysql-bin.000012
        Pos: 465
 Event_type: Query
  Server_id: 1
End_log_pos: 625
       Info: GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%'
*************************** 7. row ***************************
   Log_name: mysql-bin.000012
        Pos: 625
 Event_type: Anonymous_Gtid
  Server_id: 1
End_log_pos: 690
       Info: SET @@SESSION.GTID_NEXT= 'ANONYMOUS'
*************************** 8. row ***************************
   Log_name: mysql-bin.000012
        Pos: 690
 Event_type: Query
  Server_id: 1
End_log_pos: 777
       Info: FLUSH PRIVILEGES
8 rows in set (0.00 sec)

```

### (4). 授权Slave(Canal)连接到MySQL的账号,
```
mysql> CREATE USER canal IDENTIFIED BY 'canal';
Query OK, 0 rows affected (0.02 sec)

mysql> GRANT SELECT,REPLICATION SLAVE,REPLICATION CLIENT ON  *.* TO 'canal'@'%';
Query OK, 0 rows affected, 1 warning (0.01 sec)

mysql> FLUSH PRIVILEGES;
    Query OK, 0 rows affected (0.01 sec)


mysql> SELECT user,host FROM mysql.user;
+----------------+-----------+
| user           | host      |
+----------------+-----------+
| canal          | %         |
+----------------+-----------+
1 rows in set (0.00 sec)

```
### (5). Canal Server安装
> 省略github下载和编译过程

```
lixin-macbook:canal-server lixin$ pwd
/Users/lixin/Developer/canal-server

lixin-macbook:canal-server lixin$ tree -L 2
.
├── bin
│   ├── restart.sh
│   ├── startup.bat
│   ├── startup.sh
│   └── stop.sh
├── conf
│   ├── canal.properties
│   ├── canal_local.properties
│   ├── example
│   ├── logback.xml
│   ├── metrics
│   └── spring
├── lib
│   ├── aopalliance-1.0.jar
│   ├── aviator-2.2.1.jar
│   ├── canal.common-1.1.4.jar
│   ├── canal.deployer-1.1.4.jar
│   ├── canal.filter-1.1.4.jar
│   ├── canal.instance.core-1.1.4.jar
│   ├── canal.instance.manager-1.1.4.jar
│   ├── canal.instance.spring-1.1.4.jar
│   ├── canal.meta-1.1.4.jar
│   ├── canal.parse-1.1.4.jar
│   ├── canal.parse.dbsync-1.1.4.jar
│   ├── canal.parse.driver-1.1.4.jar
│   ├── canal.prometheus-1.1.4.jar
│   ├── canal.protocol-1.1.4.jar
│   ├── canal.server-1.1.4.jar
│   ├── canal.sink-1.1.4.jar
│   ├── canal.store-1.1.4.jar
│   ├── commons-beanutils-1.8.2.jar
│   ├── commons-cli-1.2.jar
│   ├── commons-codec-1.9.jar
│   ├── commons-compress-1.9.jar
│   ├── commons-io-2.4.jar
│   ├── commons-lang-2.6.jar
│   ├── commons-lang3-3.4.jar
│   ├── commons-logging-1.1.3.jar
│   ├── disruptor-3.4.2.jar
│   ├── druid-1.1.9.jar
│   ├── fastjson-1.2.58.jar
│   ├── fastsql-2.0.0_preview_973.jar
│   ├── guava-18.0.jar
│   ├── h2-1.4.196.jar
│   ├── httpclient-4.5.1.jar
│   ├── httpcore-4.4.3.jar
│   ├── ibatis-sqlmap-2.3.4.726.jar
│   ├── jackson-annotations-2.9.0.jar
│   ├── jackson-core-2.9.6.jar
│   ├── jackson-databind-2.9.6.jar
│   ├── javax.annotation-api-1.3.2.jar
│   ├── jcl-over-slf4j-1.7.12.jar
│   ├── jctools-core-2.1.2.jar
│   ├── jopt-simple-5.0.4.jar
│   ├── jsr305-3.0.2.jar
│   ├── kafka-clients-1.1.1.jar
│   ├── kafka_2.11-1.1.1.jar
│   ├── logback-classic-1.1.3.jar
│   ├── logback-core-1.1.3.jar
│   ├── lz4-java-1.4.1.jar
│   ├── metrics-core-2.2.0.jar
│   ├── mysql-connector-java-5.1.47.jar
│   ├── netty-3.2.2.Final.jar
│   ├── netty-all-4.1.6.Final.jar
│   ├── netty-tcnative-boringssl-static-1.1.33.Fork26.jar
│   ├── oro-2.0.8.jar
│   ├── protobuf-java-3.6.1.jar
│   ├── rocketmq-acl-4.5.2.jar
│   ├── rocketmq-client-4.5.2.jar
│   ├── rocketmq-common-4.5.2.jar
│   ├── rocketmq-logging-4.5.2.jar
│   ├── rocketmq-remoting-4.5.2.jar
│   ├── rocketmq-srvutil-4.5.2.jar
│   ├── scala-library-2.11.12.jar
│   ├── scala-logging_2.11-3.8.0.jar
│   ├── scala-reflect-2.11.12.jar
│   ├── simpleclient-0.4.0.jar
│   ├── simpleclient_common-0.4.0.jar
│   ├── simpleclient_hotspot-0.4.0.jar
│   ├── simpleclient_httpserver-0.4.0.jar
│   ├── simpleclient_pushgateway-0.4.0.jar
│   ├── slf4j-api-1.7.12.jar
│   ├── snakeyaml-1.19.jar
│   ├── snappy-java-1.1.7.1.jar
│   ├── spring-aop-3.2.18.RELEASE.jar
│   ├── spring-beans-3.2.18.RELEASE.jar
│   ├── spring-context-3.2.18.RELEASE.jar
│   ├── spring-core-3.2.18.RELEASE.jar
│   ├── spring-expression-3.2.18.RELEASE.jar
│   ├── spring-jdbc-3.2.18.RELEASE.jar
│   ├── spring-orm-3.2.18.RELEASE.jar
│   ├── spring-tx-3.2.18.RELEASE.jar
│   ├── zkclient-0.10.jar
│   └── zookeeper-3.4.5.jar
└── logs

7 directories, 88 files
```

### (6). 配置实例(instance)
```
#################################################
## mysql serverId , v1.0.26+ will autoGen
# canal.instance.mysql.slaveId=0

# enable gtid use true/false
canal.instance.gtidon=false

# position info
# MySQL地址
canal.instance.master.address=127.0.0.1:3306
canal.instance.master.journal.name=
canal.instance.master.position=
canal.instance.master.timestamp=
canal.instance.master.gtid=

# rds oss binlog
canal.instance.rds.accesskey=
canal.instance.rds.secretkey=
canal.instance.rds.instanceId=

# table meta tsdb info
canal.instance.tsdb.enable=true
#canal.instance.tsdb.url=jdbc:mysql://127.0.0.1:3306/canal_tsdb
#canal.instance.tsdb.dbUsername=canal
#canal.instance.tsdb.dbPassword=canal

#canal.instance.standby.address =
#canal.instance.standby.journal.name =
#canal.instance.standby.position =
#canal.instance.standby.timestamp =
#canal.instance.standby.gtid=

# username/password
# 用户名和密码
canal.instance.dbUsername=canal
canal.instance.dbPassword=canal
canal.instance.connectionCharset = UTF-8
# enable druid Decrypt database password
canal.instance.enableDruid=false
#canal.instance.pwdPublicKey=MFwwDQYJKoZIhvcNAQEBBQADSwAwSAJBALK4BUxdDltRRE5/zXpVEVPUgunvscYFtEip3pmLlhrWpacX7y7GCMo2/JM6LeHmiiNdH1FWgGCpUfircSwlWKUCAwEAAQ==

# table regex
canal.instance.filter.regex=.*\\..*
# table black regex
canal.instance.filter.black.regex=
# table field filter(format: schema1.tableName1:field1/field2,schema2.tableName2:field1/field2)
#canal.instance.filter.field=test1.t_product:id/subject/keywords,test2.t_company:id/name/contact/ch
# table field black filter(format: schema1.tableName1:field1/field2,schema2.tableName2:field1/field2)
#canal.instance.filter.black.field=test1.t_product:subject/product_image,test2.t_company:id/name/contact/ch

# mq config
canal.mq.topic=example
# dynamic topic route by schema or table regex
#canal.mq.dynamicTopic=mytest1.user,mytest2\\..*,.*\\..*
canal.mq.partition=0
# hash partition config
#canal.mq.partitionsNum=3
#canal.mq.partitionHash=test.table:id^name,.*\\..*
#################################################
```

### (7). 启动Canal Server
>  lixin-macbook:canal-server lixin$ ./bin/startup.sh

### (8). 查看日志是否启动成功
> lixin-macbook:canal-server lixin$ tail -500f logs/canal/canal.log 

```
2020-11-10 14:20:38.393 [main] INFO  com.alibaba.otter.canal.deployer.CanalLauncher - ## set default uncaught exception handler
2020-11-10 14:20:38.438 [main] INFO  com.alibaba.otter.canal.deployer.CanalLauncher - ## load canal configurations
2020-11-10 14:20:38.453 [main] INFO  com.alibaba.otter.canal.deployer.CanalStarter - ## start the canal server.
2020-11-10 14:20:38.499 [main] INFO  com.alibaba.otter.canal.deployer.CanalController - ## start the canal server[172.17.17.3(172.17.17.3):11111]
## canal server启动成功
2020-11-10 14:20:39.825 [main] INFO  com.alibaba.otter.canal.deployer.CanalStarter - ## the canal server is running now ......
```

> lixin-macbook:canal-server lixin$ tail -500f logs/example/example.log 

```
2020-11-10 14:20:38.945 [main] INFO  c.a.o.c.i.spring.support.PropertyPlaceholderConfigurer - Loading properties file from class path resource [canal.properties]
2020-11-10 14:20:38.950 [main] INFO  c.a.o.c.i.spring.support.PropertyPlaceholderConfigurer - Loading properties file from class path resource [example/instance.properties]
2020-11-10 14:20:39.164 [main] WARN  o.s.beans.GenericTypeAwarePropertyDescriptor - Invalid JavaBean property 'connectionCharset' being accessed! Ambiguous write methods found next to actually used [public void com.alibaba.otter.canal.parse.inbound.mysql.AbstractMysqlEventParser.setConnectionCharset(java.lang.String)]: [public void com.alibaba.otter.canal.parse.inbound.mysql.AbstractMysqlEventParser.setConnectionCharset(java.nio.charset.Charset)]
2020-11-10 14:20:39.220 [main] INFO  c.a.o.c.i.spring.support.PropertyPlaceholderConfigurer - Loading properties file from class path resource [canal.properties]
2020-11-10 14:20:39.221 [main] INFO  c.a.o.c.i.spring.support.PropertyPlaceholderConfigurer - Loading properties file from class path resource [example/instance.properties]
2020-11-10 14:20:39.747 [main] INFO  c.a.otter.canal.instance.spring.CanalInstanceWithSpring - start CannalInstance for 1-example 
2020-11-10 14:20:39.757 [main] WARN  c.a.o.canal.parse.inbound.mysql.dbsync.LogEventConvert - --> init table filter : ^.*\..*$
2020-11-10 14:20:39.757 [main] WARN  c.a.o.canal.parse.inbound.mysql.dbsync.LogEventConvert - --> init table black filter : 
2020-11-10 14:20:39.766 [main] INFO  c.a.otter.canal.instance.core.AbstractCanalInstance - start successful....
2020-11-10 14:20:39.840 [destination = example , address = /127.0.0.1:3306 , EventParser] WARN  c.a.o.c.p.inbound.mysql.rds.RdsBinlogEventParserProxy - ---> begin to find start position, it will be long time for reset or first position
2020-11-10 14:20:39.840 [destination = example , address = /127.0.0.1:3306 , EventParser] WARN  c.a.o.c.p.inbound.mysql.rds.RdsBinlogEventParserProxy - prepare to find start position just show master status
2020-11-10 14:20:40.884 [destination = example , address = /127.0.0.1:3306 , EventParser] WARN  c.a.o.c.p.inbound.mysql.rds.RdsBinlogEventParserProxy - ---> find start position successfully, EntryPosition[included=false,journalName=mysql-bin.000012,position=4,serverId=1,gtid=<null>,timestamp=1604986771000] cost : 1025ms , the next step is binlog dump
```
### (9). 关闭
> lixin-macbook:canal-server lixin$ ./bin/stop.sh 

```
lixin-macbook.local: stopping canal 5296 ... 
Oook! cost:2
```