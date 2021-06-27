---
layout: post
title: 'TiDB Docker集群安装(四)'
date: 2021-06-01
author: 李新
tags:  TiDB
---

### 1. 前言
> TiDB对硬件的要求比较高,如果是学习,建议使用:Docker集群安装.   

### 2. 下载tidb-docker-compose
> 也可参考该地址: https://docs.pingcap.com/zh/tidb/v3.0/test-deployment-using-docker

```
# 1. 下载tidb-docker-compose/
lixin-macbook:GitRepository lixin$ git clone https://github.com/pingcap/tidb-docker-compose.git

# 2. 进入tidb-docker-compose目录
lixin-macbook:GitRepository lixin$ cd tidb-docker-compose/
```

### 3. 编辑(docker-compose.yml)
> 如果你的机器配置比较高,可以忽略这一步,因为,我遇到了问题.所以,才需要配置:docker-compose.yml.   
> 去掉:tispark-master/tispark-slave0/tidb-vision/pushgateway/prometheus/grafana.   

```
# 当遇到这个问题时,可能是机器配置太低了,怎么都连接不上.
lixin-macbook:~ lixin$ mysql -h127.0.0.1   -P4000 -uroot
ERROR 2013 (HY000): Lost connection to MySQL server at 'reading initial communication packet', system error: 0
```

> 最终docker-compose.yml内容如下:

```
version: '2.1'

services:
  pd0:
    image: pingcap/pd:latest
    ports:
      - "2379"
    volumes:
      - ./config/pd.toml:/pd.toml:ro
      - ./data:/data
      - ./logs:/logs
    command:
      - --name=pd0
      - --client-urls=http://0.0.0.0:2379
      - --peer-urls=http://0.0.0.0:2380
      - --advertise-client-urls=http://pd0:2379
      - --advertise-peer-urls=http://pd0:2380
      - --initial-cluster=pd0=http://pd0:2380,pd1=http://pd1:2380,pd2=http://pd2:2380
      - --data-dir=/data/pd0
      - --config=/pd.toml
      - --log-file=/logs/pd0.log
    restart: on-failure
  pd1:
    image: pingcap/pd:latest
    ports:
      - "2379"
    volumes:
      - ./config/pd.toml:/pd.toml:ro
      - ./data:/data
      - ./logs:/logs
    command:
      - --name=pd1
      - --client-urls=http://0.0.0.0:2379
      - --peer-urls=http://0.0.0.0:2380
      - --advertise-client-urls=http://pd1:2379
      - --advertise-peer-urls=http://pd1:2380
      - --initial-cluster=pd0=http://pd0:2380,pd1=http://pd1:2380,pd2=http://pd2:2380
      - --data-dir=/data/pd1
      - --config=/pd.toml
      - --log-file=/logs/pd1.log
    restart: on-failure
  pd2:
    image: pingcap/pd:latest
    ports:
      - "2379"
    volumes:
      - ./config/pd.toml:/pd.toml:ro
      - ./data:/data
      - ./logs:/logs
    command:
      - --name=pd2
      - --client-urls=http://0.0.0.0:2379
      - --peer-urls=http://0.0.0.0:2380
      - --advertise-client-urls=http://pd2:2379
      - --advertise-peer-urls=http://pd2:2380
      - --initial-cluster=pd0=http://pd0:2380,pd1=http://pd1:2380,pd2=http://pd2:2380
      - --data-dir=/data/pd2
      - --config=/pd.toml
      - --log-file=/logs/pd2.log
    restart: on-failure
  tikv0:
    image: pingcap/tikv:latest
    volumes:
      - ./config/tikv.toml:/tikv.toml:ro
      - ./data:/data
      - ./logs:/logs
    command:
      - --addr=0.0.0.0:20160
      - --advertise-addr=tikv0:20160
      - --data-dir=/data/tikv0
      - --pd=pd0:2379,pd1:2379,pd2:2379
      - --config=/tikv.toml
      - --log-file=/logs/tikv0.log
    depends_on:
      - "pd0"
      - "pd1"
      - "pd2"
    restart: on-failure
  tikv1:
    image: pingcap/tikv:latest
    volumes:
      - ./config/tikv.toml:/tikv.toml:ro
      - ./data:/data
      - ./logs:/logs
    command:
      - --addr=0.0.0.0:20160
      - --advertise-addr=tikv1:20160
      - --data-dir=/data/tikv1
      - --pd=pd0:2379,pd1:2379,pd2:2379
      - --config=/tikv.toml
      - --log-file=/logs/tikv1.log
    depends_on:
      - "pd0"
      - "pd1"
      - "pd2"
    restart: on-failure
  tikv2:
    image: pingcap/tikv:latest
    volumes:
      - ./config/tikv.toml:/tikv.toml:ro
      - ./data:/data
      - ./logs:/logs
    command:
      - --addr=0.0.0.0:20160
      - --advertise-addr=tikv2:20160
      - --data-dir=/data/tikv2
      - --pd=pd0:2379,pd1:2379,pd2:2379
      - --config=/tikv.toml
      - --log-file=/logs/tikv2.log
    depends_on:
      - "pd0"
      - "pd1"
      - "pd2"
    restart: on-failure

  tidb:
    image: pingcap/tidb:latest
    ports:
      - "4000:4000"
      - "10080:10080"
    volumes:
      - ./config/tidb.toml:/tidb.toml:ro
      - ./logs:/logs
    command:
      - --store=tikv
      - --path=pd0:2379,pd1:2379,pd2:2379
      - --config=/tidb.toml
      - --log-file=/logs/tidb.log
      - --advertise-address=tidb
    depends_on:
      - "tikv0"
      - "tikv1"
      - "tikv2"
    restart: on-failure
```
### 4. 安装
```
lixin-macbook:tidb-docker-compose lixin$ docker-compose pull && docker-compose up -d
Pulling pd0   ... done
Pulling pd1   ... done
Pulling pd2   ... done
Pulling tikv0 ... done
Pulling tikv1 ... done
Pulling tikv2 ... done
Pulling tidb  ... done
Docker Compose is now in the Docker CLI, try `docker compose up`

WARNING: Found orphan containers (tidb-docker-compose_tispark-slave0_1, tidb-docker-compose_tidb-vision_1, tidb-docker-compose_grafana_1, tidb-docker-compose_prometheus_1, tidb-docker-compose_tispark-master_1, tidb-docker-compose_pushgateway_1) for this project. If you removed or renamed this service in your compose file, you can run this command with the --remove-orphans flag to clean it up.
tidb-docker-compose_pd1_1 is up-to-date
tidb-docker-compose_pd2_1 is up-to-date
tidb-docker-compose_pd0_1 is up-to-date
tidb-docker-compose_tikv0_1 is up-to-date
tidb-docker-compose_tikv2_1 is up-to-date
tidb-docker-compose_tikv1_1 is up-to-date
tidb-docker-compose_tidb_1 is up-to-date
```
### 5. 通过docker-compose查看运行实例
```
lixin-macbook:tidb-docker-compose lixin$ docker-compose ps
           Name                          Command               State                        Ports
-----------------------------------------------------------------------------------------------------------------------
tidb-docker-compose_pd0_1     /pd-server --name=pd0 --cl ...   Up      0.0.0.0:50065->2379/tcp, 2380/tcp
tidb-docker-compose_pd1_1     /pd-server --name=pd1 --cl ...   Up      0.0.0.0:50064->2379/tcp, 2380/tcp
tidb-docker-compose_pd2_1     /pd-server --name=pd2 --cl ...   Up      0.0.0.0:50067->2379/tcp, 2380/tcp
tidb-docker-compose_tidb_1    /tidb-server --store=tikv  ...   Up      0.0.0.0:10080->10080/tcp, 0.0.0.0:4000->4000/tcp
tidb-docker-compose_tikv0_1   /tikv-server --addr=0.0.0. ...   Up      20160/tcp
tidb-docker-compose_tikv1_1   /tikv-server --addr=0.0.0. ...   Up      20160/tcp
tidb-docker-compose_tikv2_1   /tikv-server --addr=0.0.0. ...   Up      20160/tcp
```
### 6. 测试
```
lixin-macbook:tidb-docker-compose lixin$ mysql -h 127.0.0.1 -P 4000 -u root
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 5
Server version: 5.7.25-TiDB-v5.0.1 TiDB Server (Apache License 2.0) Community Edition, MySQL 5.7 compatible

Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| INFORMATION_SCHEMA |
| METRICS_SCHEMA     |
| PERFORMANCE_SCHEMA |
| mysql              |
| test               |
+--------------------+

# 进入测试库
mysql>  USE test;

# 创建表
mysql>  CREATE TABLE t_test (
  id int(11) NOT NULL,
  name varchar(20) DEFAULT NULL,
  age int(11) DEFAULT NULL,
  score decimal(10,2) DEFAULT NULL,
  PRIMARY KEY (id)  ,
  KEY idx_name (name),
  KEY idx_score (score)
);

# 插入数据
INSERT INTO t_test(id,name,age,score) VALUES(1,"张三",18,59.9);
INSERT INTO t_test(id,name,age,score) VALUES(2,"李四",18,58.9);
INSERT INTO t_test(id,name,age,score) VALUES(3,"王五",18,57.9);
INSERT INTO t_test(id,name,age,score) VALUES(4,"赵六",18,69.9);
INSERT INTO t_test(id,name,age,score) VALUES(5,"张三丰",18,66.9);
INSERT INTO t_test(id,name,age,score) VALUES(6,"赵敏",18,70.9);
INSERT INTO t_test(id,name,age,score) VALUES(7,"李元霸",18,79.9);
INSERT INTO t_test(id,name,age,score) VALUES(8,"李世民",18,89.9);
INSERT INTO t_test(id,name,age,score) VALUES(9,"张翠山",18,99.9);
INSERT INTO t_test(id,name,age,score) VALUES(10,"赵六",18,78.9);


# LIKE全表扫描(证明:不支持LIKE)
mysql> EXPLAIN SELECT * FROM t_test WHERE name LIKE '%三%';
+-------------------------+---------+-----------+---------------+-------------------------------------+
| id                      | estRows | task      | access object | operator info                       |
+-------------------------+---------+-----------+---------------+-------------------------------------+
| TableReader_7           | 8.00    | root      |               | data:Selection_6                    |
| └─Selection_6           | 8.00    | cop[tikv] |               | like(test.t_test.name, "%三%", 92)  |
|   └─TableFullScan_5     | 10.00   | cop[tikv] | table:t_test  | keep order:false, stats:pseudo      |
+-------------------------+---------+-----------+---------------+-------------------------------------+

mysql> EXPLAIN SELECT * FROM t_test WHERE name LIKE '%张';
+-------------------------+---------+-----------+---------------+------------------------------------+
| id                      | estRows | task      | access object | operator info                      |
+-------------------------+---------+-----------+---------------+------------------------------------+
| TableReader_7           | 8.00    | root      |               | data:Selection_6                   |
| └─Selection_6           | 8.00    | cop[tikv] |               | like(test.t_test.name, "%张", 92)  |
|   └─TableFullScan_5     | 10.00   | cop[tikv] | table:t_test  | keep order:false, stats:pseudo     |
+-------------------------+---------+-----------+---------------+------------------------------------+


# 范围扫描(IndexRangeScan/TableRowIDScan),证明用到了索引.
mysql> EXPLAIN SELECT * FROM t_test WHERE score > 60 AND score < 70;
+-------------------------------+---------+-----------+--------------------------------------+-----------------------------------------------------+
| id                            | estRows | task      | access object                        | operator info                                       |
+-------------------------------+---------+-----------+--------------------------------------+-----------------------------------------------------+
| IndexLookUp_10                | 0.25    | root      |                                      |                                                     |
| ├─IndexRangeScan_8(Build)     | 0.25    | cop[tikv] | table:t_test, index:idx_score(score) | range:(60.00,70.00), keep order:false, stats:pseudo |
| └─TableRowIDScan_9(Probe)     | 0.25    | cop[tikv] | table:t_test                         | keep order:false, stats:pseudo                      |
+-------------------------------+---------+-----------+--------------------------------------+-----------------------------------------------------+
```