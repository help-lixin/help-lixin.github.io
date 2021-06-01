---
layout: post
title: 'TiDB 单机安装(二)'
date: 2021-06-01
author: 李新
tags:  TiDB
---

### 1. 环境要求
> TiDB单机版安装要求至少centos7以上的版本,如果是centos6版本会报版本不匹配错误.

### 2. TiDB安装
```
# 1. 下载最新版本
[root@clickhouse-1 soft]# wget http://download.pingcap.org/tidb-latest-linux-amd64.tar.gz

# 2. 检查下文件
[root@clickhouse-1 soft]# ll|grep tidb-latest-linux-amd64.tar.gz
-rw-r--r-- 1 root root 517255312 Apr 24 21:58 tidb-latest-linux-amd64.tar.gz

# 3. 解压
[root@clickhouse-1 soft]# tar -zxvf tidb-latest-linux-amd64.tar.gz


# 4. 进入解压后的tidb目录
[root@clickhouse-1 soft]# cd tidb-v5.0.1-linux-amd64/

# 5. 查看目录结构
[root@clickhouse-1 tidb-v5.0.1-linux-amd64]# tree
.
├── bin
│   ├── arbiter
│   ├── binlogctl
│   ├── drainer
│   ├── etcdctl
│   ├── pd-ctl
│   ├── pd-recover
│   ├── pd-server
│   ├── pump
│   ├── reparo
│   ├── tidb-ctl
│   ├── tidb-server
│   ├── tikv-ctl
│   └── tikv-server
├── PingCAP\ Community\ Software\ Agreement(Chinese\ Version).pdf
└── PingCAP\ Community\ Software\ Agreement(English\ Version).pdf
```
### 3. 启动PD
```
# 1.启动pd(会监听:2379/2380端口)
[root@clickhouse-1 tidb-v5.0.1-linux-amd64]# ./bin/pd-server  --data-dir=pd --log-file=pd.log &

# 2. 生成了pd目录和pd.log文件
[root@clickhouse-1 tidb-v5.0.1-linux-amd64]# ll
drwxr-xr-x 2 root root    4096 Apr 24 10:12 bin
drwx------ 4 root root    4096 May 31 20:09 pd
-rw-r--r-- 1 root root   15174 May 31 20:09 pd.log
-rw-r--r-- 1 root root 1009394 Apr 16  2020 PingCAP Community Software Agreement(Chinese Version).pdf
-rw-r--r-- 1 root root  158646 Apr 16  2020 PingCAP Community Software Agreement(English Version).pdf
```
### 4. 启动tikv
```
# 启动tikv(会监听:20180/20160)
[root@clickhouse-1 tidb-v5.0.1-linux-amd64]# ./bin/tikv-server --pd="127.0.0.1:2379"  --data-dir=tikv  --log-file=tikv.log &

```
### 5. 启动tidb
```
# 启动tidb(4000端口)
[root@clickhouse-1 tidb-v5.0.1-linux-amd64]# ./bin/tidb-server  --store=tikv  --path="127.0.0.1:2379"  --log-file=tidb.log &  
```
### 6. 测试
```
lixin-macbook:~ lixin$ mysql -h  10.211.55.100 -P 4000 -u root
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 7
*******************************************************************
Server version: 5.7.25-TiDB-v5.0.1 TiDB Server (Apache License 2.0) Community Edition, MySQL 5.7 compatible
*******************************************************************

Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```
### 7. 总结
> 
