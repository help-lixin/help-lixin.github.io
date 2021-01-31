---
layout: post
title: 'Seata AT模式之入门案例(六)'
date: 2021-01-28
author: 李新
tags: Seata
---

### (1). springcloud-eureka-seata项目下载并编译
```
$ git clone https://github.com/seata/seata-samples.git
$ cd seata-samples
$ mvn clean install -DskipTests
```

> https://github.com/seata/seata-samples/tree/master/springcloud-eureka-seata

### (2). 导入springcloud-eureka-seata到工程
!["springcloud-eureka-seata技术架构图"](/assets/seata/imgs/seata-at-demo-architecture.png)

!["springcloud-eureka-seata项目调用链路图"](/assets/seata/imgs/seata-﻿springcloud-eureka-sequence.jpg)

```
# 查看:springcloud-eureka-seata项目目录
springcloud-eureka-seata
├── pom.xml
├── all.sql
│
├── eureka                              # 注册中心
│   ├── pom.xml
│   ├── src
│   └── target
├── bussiness                           # 业务服务(组合其它服务的一个中间服务)
│   ├── pom.xml
│   ├── src
│   └── target
├── account                             # 帐户服务:从用户帐户中扣除余额.
│   ├── pom.xml
│   ├── src
│   └── target
├── order                               # 订单服务:根据采购需求创建订单
│   ├── pom.xml
│   ├── src
│   └── target
└── storage                             # 仓储服务:对给定的商品扣除仓储数量.
    ├── pom.xml
    ├── src
    └── target
```
### (3). 导入sql脚本
> springcloud-eureka-seata/all.sql

```
$ mysql -u root -p

# 创建fescar库
mysql> create database fescar;
Query OK, 1 row affected (0.01 sec)

mysql> use fescar;
Database changed

# 导入脚本
mysql> source /Users/lixin/GitRepository/seata-samples/springcloud-eureka-seata/all.sql

# 查看表信息
mysql> show tables;
+------------------+
| Tables_in_fescar |
+------------------+
| account_tbl      |     # 账户表
| order_tbl        |     # 订单表
| storage_tbl      |     # 仓储表
| undo_log         |     # undo日志表
+------------------+
```
### (4). 配置(storage/order/account/bussiness)数据源
> application.properties

```
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/fescar?useSSL=false&serverTimezone=UTC
spring.datasource.username=root
spring.datasource.password=123456
```
### (5). 启动Seata Server(TC[事务协调器])

> 1. 配置registry.conf(配置中心为:eureka),注意:TODO,是我改动的地方

```
lixin-macbook:seata-server-1.4.0 lixin$ cat conf/registry.conf
registry {
  # file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
  # type = "file"
  # TODO lixin
  type = "eureka"
  loadBalance = "RandomLoadBalance"
  loadBalanceVirtualNodes = 10

  nacos {
    application = "seata-server"
    serverAddr = "127.0.0.1:8848"
    group = "SEATA_GROUP"
    namespace = ""
    cluster = "default"
    username = ""
    password = ""
  }
  eureka {
    serviceUrl = "http://localhost:8761/eureka"
    application = "default"
    weight = "1"
  }
  redis {
    serverAddr = "localhost:6379"
    db = 0
    password = ""
    cluster = "default"
    timeout = 0
  }
  zk {
    cluster = "default"
    serverAddr = "127.0.0.1:2181"
    sessionTimeout = 6000
    connectTimeout = 2000
    username = ""
    password = ""
  }
  consul {
    cluster = "default"
    serverAddr = "127.0.0.1:8500"
  }
  etcd3 {
    cluster = "default"
    serverAddr = "http://localhost:2379"
  }
  sofa {
    serverAddr = "127.0.0.1:9603"
    application = "default"
    region = "DEFAULT_ZONE"
    datacenter = "DefaultDataCenter"
    cluster = "default"
    group = "SEATA_GROUP"
    addressWaitTime = "3000"
  }
  file {
    name = "file.conf"
  }
}

config {
  # file、nacos 、apollo、zk、consul、etcd3
  type = "file"

  nacos {
    serverAddr = "127.0.0.1:8848"
    namespace = ""
    group = "SEATA_GROUP"
    username = ""
    password = ""
  }
  consul {
    serverAddr = "127.0.0.1:8500"
  }
  apollo {
    appId = "seata-server"
    apolloMeta = "http://192.168.1.204:8801"
    namespace = "application"
    apolloAccesskeySecret = ""
  }
  zk {
    serverAddr = "127.0.0.1:2181"
    sessionTimeout = 6000
    connectTimeout = 2000
    username = ""
    password = ""
  }
  etcd3 {
    serverAddr = "http://localhost:2379"
  }
  file {
    name = "file.conf"
  }
}
```

> 2. 配置file.conf 
> 配置数据存储模式为db,注意:TODO,是我改动的地方.   
> 我本来用的是file模式,在tc日志里,也能看到已经注册了XID,可是到了分支事务里,却发现:XID不存在,估计是磁盘写入与分支事务之间存在时间差异,所以我改成了:db模式.    

```
lixin-macbook:seata-server-1.4.0 lixin$ cat conf/file.conf
## transaction log store, only used in seata-server
store {
  ## store mode: file、db、redis
  // TODO lixin 数据存储模式改为:db
  # mode = "file"
  mode = "db"

  ## file store property
  file {
    ## store location dir
    dir = "sessionStore"
    # branch session size , if exceeded first try compress lockkey, still exceeded throws exceptions
    maxBranchSessionSize = 16384
    # globe session size , if exceeded throws exceptions
    maxGlobalSessionSize = 512
    # file buffer size , if exceeded allocate new buffer
    fileWriteBufferCacheSize = 16384
    # when recover batch read size
    sessionReloadReadSize = 100
    # async, sync
    flushDiskMode = async
  }

  ## database store property
  db {
    ## the implement of javax.sql.DataSource, such as DruidDataSource(druid)/BasicDataSource(dbcp)/HikariDataSource(hikari) etc.
    datasource = "druid"
    ## mysql/oracle/postgresql/h2/oceanbase etc.
    dbType = "mysql"
    driverClassName = "com.mysql.jdbc.Driver"
    url = "jdbc:mysql://127.0.0.1:3306/seata"
    user = "root" #TODO lixin
    password = "123456" #TODO lixin
    minConn = 5
    maxConn = 100
    globalTable = "global_table"
    branchTable = "branch_table"
    lockTable = "lock_table"
    queryLimit = 100
    maxWait = 5000
  }

  ## redis store property
  redis {
    host = "127.0.0.1"
    port = "6379"
    password = ""
    database = "0"
    minConn = 1
    maxConn = 10
    maxTotal = 100
    queryLimit = 100
  }

}
```

> 3. 创建库和表结构.  
> ["脚本下载地址"](https://github.com/seata/seata/blob/develop/script/server/db/mysql.sql)    
> 创建seata库,并导入脚本.   

```
-- -------------------------------- The script used when storeMode is 'db' --------------------------------
-- the table to store GlobalSession data
CREATE TABLE IF NOT EXISTS `global_table`
(
    `xid`                       VARCHAR(128) NOT NULL,
    `transaction_id`            BIGINT,
    `status`                    TINYINT      NOT NULL,
    `application_id`            VARCHAR(32),
    `transaction_service_group` VARCHAR(32),
    `transaction_name`          VARCHAR(128),
    `timeout`                   INT,
    `begin_time`                BIGINT,
    `application_data`          VARCHAR(2000),
    `gmt_create`                DATETIME,
    `gmt_modified`              DATETIME,
    PRIMARY KEY (`xid`),
    KEY `idx_gmt_modified_status` (`gmt_modified`, `status`),
    KEY `idx_transaction_id` (`transaction_id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8;

-- the table to store BranchSession data
CREATE TABLE IF NOT EXISTS `branch_table`
(
    `branch_id`         BIGINT       NOT NULL,
    `xid`               VARCHAR(128) NOT NULL,
    `transaction_id`    BIGINT,
    `resource_group_id` VARCHAR(32),
    `resource_id`       VARCHAR(256),
    `branch_type`       VARCHAR(8),
    `status`            TINYINT,
    `client_id`         VARCHAR(64),
    `application_data`  VARCHAR(2000),
    `gmt_create`        DATETIME(6),
    `gmt_modified`      DATETIME(6),
    PRIMARY KEY (`branch_id`),
    KEY `idx_xid` (`xid`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8;

-- the table to store lock data
CREATE TABLE IF NOT EXISTS `lock_table`
(
    `row_key`        VARCHAR(128) NOT NULL,
    `xid`            VARCHAR(96),
    `transaction_id` BIGINT,
    `branch_id`      BIGINT       NOT NULL,
    `resource_id`    VARCHAR(256),
    `table_name`     VARCHAR(32),
    `pk`             VARCHAR(36),
    `gmt_create`     DATETIME,
    `gmt_modified`   DATETIME,
    PRIMARY KEY (`row_key`),
    KEY `idx_branch_id` (`branch_id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8;
```

> 4. 拷贝mysql jar包到lib目录下(略)   

> 5. 启动Seata Server.   

```
# 可以自行下载,也可以通过我前面编译后产生的(seata-v1.4.0/distribution/target/seata-server-1.4.0.tar.gz).
lixin-macbook:seata-server-1.4.0 lixin$ ./bin/seata-server.sh -m db
15:35:19.083  INFO --- [main] i.s.core.rpc.netty.NettyServerBootstrap  : Server started, listen port: 8091
```
### (6). 启动项目(略)
> 注意: 先要启动eureka.

### (7). 查看数据信息
> 在BusinessService类实例化时,会同时往(account_tbl/storage_tbl)表中增加一些测试数据.  

```
mysql> use fescar;
Database changed

# 帐户表(account_tbl),用户:U100000,余额为:10000
mysql> select * from account_tbl;
+----+---------+-------+
| id | user_id | money |
+----+---------+-------+
| 1  | U100000 | 10000 |
+----+---------+-------+
1 row in set (0.00 sec)

mysql> select * from order_tbl;
Empty set (0.00 sec)

# 仓库表(storage_tbl),商品ID:C100000的库存数量为:200
mysql> select * from storage_tbl;
+----+----------------+-------+
| id | commodity_code | count |
+----+----------------+-------+
| 1  | C100000        |   200 |
+----+----------------+-------+
1 row in set (0.00 sec)

mysql> select * from undo_log;
Empty set (0.00 sec)
```
### (8). 测试正常情况
```
# 发起请求
$ curl http://localhost:8084/purchase/commit

# 检查表
# 1. 库存(storage_tbl)要减少30为170
mysql> select * from storage_tbl;
+----+----------------+-------+
| id | commodity_code | count |
+----+----------------+-------+
|  1 | C100000        |   170 |
+----+----------------+-------+
1 row in set (0.00 sec)

# 2. 订单要增加(order_tbl)
mysql> select * from order_tbl;
+----+---------+----------------+-------+-------+
| id | user_id | commodity_code | count | money |
+----+---------+----------------+-------+-------+
|  2 | U100000 | C100000        |    30 |  3000 |
+----+---------+----------------+-------+-------+
1 row in set (0.00 sec)

# 3. 帐户余额(account_tbl)要减少300为:7000
mysql> select * from account_tbl;
+----+---------+-------+
| id | user_id | money |
+----+---------+-------+
|  1 | U100000 |  7000 |
+----+---------+-------+
1 row in set (0.01 sec)


# 4. 查看seata server(tc)日志
20:07:33.264  INFO --- [ettyServerNIOWorker_1_3_8] i.s.c.r.processor.server.RegTmProcessor  : TM register success,message:RegisterTMRequest{applicationId='business-service', transactionServiceGroup='my_test_tx_group'},channel:[id: 0xff43628f, L:/192.168.0.144:8091 - R:/192.168.0.144:50322],client version:1.4.0
20:07:33.303  INFO --- [     batchLoggerPrint_1_1] i.s.c.r.p.server.BatchLogHandler         : SeataMergeMessage timeout=60000,transactionName=purchase(java.lang.String, java.lang.String, int)
,clientIp:192.168.0.144,vgroup:my_test_tx_group
20:07:33.320  INFO --- [verHandlerThread_1_41_500] i.s.s.coordinator.DefaultCoordinator     : Begin new global transaction applicationId: business-service,transactionServiceGroup: my_test_tx_group, transactionName: purchase(java.lang.String, java.lang.String, int),timeout:60000,xid:192.168.0.144:8091:98511002756775936
20:07:33.673  INFO --- [     batchLoggerPrint_1_1] i.s.c.r.p.server.BatchLogHandler         : SeataMergeMessage xid=192.168.0.144:8091:98511002756775936,branchType=AT,resourceId=jdbc:mysql://127.0.0.1:3306/fescar,lockKey=storage_tbl:1
,clientIp:192.168.0.144,vgroup:my_test_tx_group
20:07:33.709  INFO --- [verHandlerThread_1_42_500] i.seata.server.coordinator.AbstractCore  : Register branch successfully, xid = 192.168.0.144:8091:98511002756775936, branchId = 98511004317057025, resourceId = jdbc:mysql://127.0.0.1:3306/fescar ,lockKeys = storage_tbl:1
20:07:33.799  INFO --- [     batchLoggerPrint_1_1] i.s.c.r.p.server.BatchLogHandler         : SeataMergeMessage xid=192.168.0.144:8091:98511002756775936,branchType=AT,resourceId=jdbc:mysql://127.0.0.1:3306/fescar,lockKey=order_tbl:2
,clientIp:192.168.0.144,vgroup:my_test_tx_group
20:07:33.838  INFO --- [verHandlerThread_1_43_500] i.seata.server.coordinator.AbstractCore  : Register branch successfully, xid = 192.168.0.144:8091:98511002756775936, branchId = 98511004841345025, resourceId = jdbc:mysql://127.0.0.1:3306/fescar ,lockKeys = order_tbl:2
20:07:33.898  INFO --- [     batchLoggerPrint_1_1] i.s.c.r.p.server.BatchLogHandler         : SeataMergeMessage xid=192.168.0.144:8091:98511002756775936,branchType=AT,resourceId=jdbc:mysql://127.0.0.1:3306/fescar,lockKey=account_tbl:1
,clientIp:192.168.0.144,vgroup:my_test_tx_group
20:07:33.932  INFO --- [verHandlerThread_1_44_500] i.seata.server.coordinator.AbstractCore  : Register branch successfully, xid = 192.168.0.144:8091:98511002756775936, branchId = 98511005256581121, resourceId = jdbc:mysql://127.0.0.1:3306/fescar ,lockKeys = account_tbl:1
20:07:33.985  INFO --- [     batchLoggerPrint_1_1] i.s.c.r.p.server.BatchLogHandler         : SeataMergeMessage xid=192.168.0.144:8091:98511002756775936,extraData=null
,clientIp:192.168.0.144,vgroup:my_test_tx_group
20:07:34.322  INFO --- [      AsyncCommitting_1_1] io.seata.server.coordinator.DefaultCore  : Committing global transaction is successfully done, xid = 192.168.0.144:8091:98511002756775936.
```

### (9). 异常测试
> 关闭account微服务

```
# 发起请求
$ curl http://localhost:8084/purchase/commit
OrderFeignClient#create(String,String,Integer) failed and no fallback available.

# 检查表
# 1. 库存(storage_tbl)要减少30为170
mysql> select * from storage_tbl;
+----+----------------+-------+
| id | commodity_code | count |
+----+----------------+-------+
|  1 | C100000        |   170 |
+----+----------------+-------+
1 row in set (0.00 sec)


# 2. 订单要增加(order_tbl)
mysql> select * from order_tbl;
+----+---------+----------------+-------+-------+
| id | user_id | commodity_code | count | money |
+----+---------+----------------+-------+-------+
|  2 | U100000 | C100000        |    30 |  3000 |
+----+---------+----------------+-------+-------+
1 row in set (0.00 sec)

# 3. 帐户余额(account_tbl)要减少300为:7000
mysql> select * from account_tbl;
+----+---------+-------+
| id | user_id | money |
+----+---------+-------+
|  1 | U100000 |  7000 |
+----+---------+-------+
1 row in set (0.00 sec)
```
### (10). 再次正常测试
> 开启account微服务,再次正常测试,因为表(order_tbl)ID是自增的,所以,刚才失败的ID按理应该是要跳过.

```
# 发起请求
$ curl http://localhost:8084/purchase/commit

# 检查表
# 1. 库存(storage_tbl)要减少30为170
mysql> select * from storage_tbl;
+----+----------------+-------+
| id | commodity_code | count |
+----+----------------+-------+
|  1 | C100000        |   140 |
+----+----------------+-------+
1 row in set (0.00 sec)

# 2. 订单要增加(order_tbl)
# 我这里不刚才不小心又创建了一次失败的,所以ID变成了:5
mysql> select * from order_tbl;
+----+---------+----------------+-------+-------+
| id | user_id | commodity_code | count | money |
+----+---------+----------------+-------+-------+
|  2 | U100000 | C100000        |    30 |  3000 |
|  5 | U100000 | C100000        |    30 |  3000 |
+----+---------+----------------+-------+-------+
2 rows in set (0.00 sec)

# 3. 帐户余额(account_tbl)要减少300为:7000
mysql> select * from account_tbl;
+----+---------+-------+
| id | user_id | money |
+----+---------+-------+
|  1 | U100000 |  4000 |
+----+---------+-------+
1 row in set (0.00 sec)

```
### (11). 总结
> 通过Seata AT模式,确实可以保证了数据的最终一致性.   