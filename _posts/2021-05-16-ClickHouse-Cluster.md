---
layout: post
title: 'ClickHouse 集群安装(二)'
date: 2021-05-16
author: 李新
tags:  ClickHouse
---

### (1). 机器配置

|  hostname     | ip            |
|  ----         | ----          |
| clickhouse-1  | 10.211.55.100 |
| clickhouse-2  | 10.211.55.101 |
| clickhouse-3  | 10.211.55.102 |

### (2). 安装前准备(略)

### (3). ZK集群(略)

### (4). RPM包下载
> 需要下载:clickhouse-client/clickhouse-common-static/clickhouse-server/clickhouse-server-common.   

["RPM包下载"](https://packagecloud.io/Altinity/clickhouse)  

### (5). 安装
```
# 1. 查看下载的RPM包
# ll /opt/soft/
total 103412
-rw-r--r-- 1 root root     6384 May 21 15:10 clickhouse-client-20.8.3.18-1.el7.x86_64.rpm
-rw-r--r-- 1 root root 69093220 May 21 15:10 clickhouse-common-static-20.8.3.18-1.el7.x86_64.rpm
-rw-r--r-- 1 root root 36772044 May 21 15:10 clickhouse-server-20.8.3.18-1.el7.x86_64.rpm
-rw-r--r-- 1 root root    14472 May 21 15:10 clickhouse-server-common-20.8.3.18-1.el7.x86_64.rpm

# 2. 所有的机器,都要安装rpm
# rpm -ivh *.rpm
Preparing...                          ################################# [100%]
Updating / installing...
   1:clickhouse-server-common-20.8.3.1################################# [ 25%]
   2:clickhouse-common-static-20.8.3.1################################# [ 50%]
   3:clickhouse-server-20.8.3.18-1.el7################################# [ 75%]
Create user clickhouse.clickhouse with datadir /var/lib/clickhouse
   4:clickhouse-client-20.8.3.18-1.el7################################# [100%]
Create user clickhouse.clickhouse with datadir /var/lib/clickhouse

```
### (6). 配置
> 1. 去掉注释(listen_host),不限制IP地址可访问.  

```
# 1. 开启:<listen_host>,不限制IP地址.
# 配置:/etc/clickhouse-server/config.xml
	<listen_host>::</listen_host>
```

> 2. 要注意:replica的值要在整个集群内唯一.

```
# 2. 创建/etc/metrika.xml文件
# 注意:<macros> <replica>clickhouse-1</replica> </macros>,请保下在集群内的唯一.

<yandex>
    <clickhouse_remote_servers>
        <perftest_3shards_1replicas>
            <shard>
                <internal_replication>true</internal_replication>
                <replica>
                    <host>clickhouse-1</host>
                    <port>9000</port>
                </replica>
            </shard>
            <shard>
                <replica>
                    <internal_replication>true</internal_replication>
                    <host>clickhouse-2</host>
                    <port>9000</port>
                </replica>
            </shard>
            <shard>
                <internal_replication>true</internal_replication>
                <replica>
                    <host>clickhouse-3</host>
                    <port>9000</port>
                </replica>
            </shard>
        </perftest_3shards_1replicas>
    </clickhouse_remote_servers>
     
    <zookeeper-servers>
        <node index="1">
            <host>clickhouse-1</host>
            <port>2181</port>
        </node>
    </zookeeper-servers>
    <macros>
        <replica>clickhouse-1</replica>
    </macros>
    <networks>
        <ip>::/0</ip>
    </networks>
    <clickhouse_compression>
        <case>
            <min_part_size>10000000000</min_part_size>
            <min_part_size_ratio>0.01</min_part_size_ratio>
            <method>lz4</method>
        </case>
    </clickhouse_compression>
</yandex>
```
### (7). 启动clickhouse-server
```
# 5. 启动clickhouse-server
# service clickhouse-server start
Start clickhouse-server service: Path to data directory in /etc/clickhouse-server/config.xml: /var/lib/clickhouse/
DONE
```
### (8). 验证集群
```
# 1. 通过client进行访问
[root@clickhouse-1 ~]# clickhouse-client -m
ClickHouse client version 20.8.3.18.
Connecting to localhost:9000 as user default.
Connected to ClickHouse server version 20.8.3 revision 54438.

# 2. 检查clusters表
clickhouse-1 :) SELECT * FROM system.clusters;
┌─cluster───────────────────────────┬─shard_num─┬─shard_weight─┬─replica_num─┬─host_name────┬─host_address──┬─port─┬─is_local─┬─user────┬─default_database─┬─errors_count─┬─estimated_recovery_time─┐
│ perftest_3shards_1replicas        │         1 │            1 │           1 │ clickhouse-1 │ 10.211.55.100 │ 9000 │        1 │ default │                  │            0 │                       0 │
│ perftest_3shards_1replicas        │         2 │            1 │           1 │ clickhouse-2 │ 10.211.55.101 │ 9000 │        0 │ default │                  │            0 │                       0 │
│ perftest_3shards_1replicas        │         3 │            1 │           1 │ clickhouse-3 │ 10.211.55.102 │ 9000 │        0 │ default │                  │            0 │                       0 │
└───────────────────────────────────┴───────────┴──────────────┴─────────────┴──────────────┴───────────────┴──────┴──────────┴─────────┴──────────────────┴──────────────┴─────────────────────────┘
```
### (9). 总结