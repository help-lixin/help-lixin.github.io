---
layout: post
title: 'Etcd 集群搭建'
date: 2021-01-01
author: 李新
tags: Etcd
---

### (1). 机器准备

|     IP         | 主机名称     |
|  ----          |   ----      |
| 10.211.55.100  |  master     |
| 10.211.55.101  |  node-1     |
| 10.211.55.102  |  node-2     |

### (2). 所有机器关闭防火墙和selinux
```
# 所有机器关闭防火墙
$ systemctl stop firewalld
$ systemctl disable firewalld

# 所有机器关闭selinux
$ sed -i 's/enforcing/disabled/' /etc/selinux/config 
$ setenforce 0
```
### (3). 下载etcd
```
# 下载etcd
$ wget https://github.com/etcd-io/etcd/releases/download/v3.4.14/etcd-v3.4.14-linux-amd64.tar.gz

# 解压
$ tar -xvf etcd-v3.4.14-linux-amd64.tar.gz

# 移动etcd*可执行文件到PATH目录下
$ mv etcd-v3.4.14-linux-amd64/etcd* /usr/local/bin/


# 创建etcd的配置文件目录和数据文件目录
# /etc/etcd     : 为配置文件目录
# /var/lib/etcd : 为数据文件目录
$ mkdir -p /etc/etcd  /var/lib/etcd
```
### (4). 配置(/etc/etcd/etcd-cfg.yml)
> ["Etcd参考博客"](https://www.jianshu.com/p/b399cb3c2516)

> master配置(/etc/etcd/etcd-cfg.yml)

```
name: etcd-0
data-dir: /var/lib/etcd
listen-client-urls: http://10.211.55.100:2379,http://127.0.0.1:2379
advertise-client-urls: http://10.211.55.100:2379,http://127.0.0.1:2379
listen-peer-urls: http://10.211.55.100:2380
initial-advertise-peer-urls: http://10.211.55.100:2380
initial-cluster: etcd-0=http://10.211.55.100:2380,etcd-1=http://10.211.55.101:2380,etcd-2=http://10.211.55.102:2380
initial-cluster-token: etcd-cluster-token
initial-cluster-state: new
```

> node-1配置(/etc/etcd/etcd-cfg.yml)

```
name: etcd-1
data-dir: /var/lib/etcd
listen-client-urls: http://10.211.55.101:2379,http://127.0.0.1:2379
advertise-client-urls: http://10.211.55.101:2379,http://127.0.0.1:2379
listen-peer-urls: http://10.211.55.101:2380
initial-advertise-peer-urls: http://10.211.55.101:2380
initial-cluster: etcd-0=http://10.211.55.100:2380,etcd-1=http://10.211.55.101:2380,etcd-2=http://10.211.55.102:2380
initial-cluster-token: etcd-cluster-token
initial-cluster-state: new
```

> node-2配置(/etc/etcd/etcd-cfg.yml)

```
name: etcd-2
data-dir: /var/lib/etcd
listen-client-urls: http://10.211.55.102:2379,http://127.0.0.1:2379
advertise-client-urls: http://10.211.55.102:2379,http://127.0.0.1:2379
listen-peer-urls: http://10.211.55.102:2380
initial-advertise-peer-urls: http://10.211.55.102:2380
initial-cluster: etcd-0=http://10.211.55.100:2380,etcd-1=http://10.211.55.101:2380,etcd-2=http://10.211.55.102:2380
initial-cluster-token: etcd-cluster-token
initial-cluster-state: new
```

### (5). 启动etcd集群
```
[root@master ~]# etcd --config-file=/etc/etcd/etcd-cfg.yml
[root@node-1 ~]# etcd --config-file=/etc/etcd/etcd-cfg.yml
[root@node-2 ~]# etcd --config-file=/etc/etcd/etcd-cfg.yml
```
### (6). 查看集群成员
```
# 查看集群成员列表
[root@node-2 ~]# etcdctl --endpoints=10.211.55.100:2379,10.211.55.101:2379,10.211.55.102:2379 member list -w table
+------------------+---------+--------+---------------------------+-------------------------------------------------+------------+
|        ID        | STATUS  |  NAME  |        PEER ADDRS         |                  CLIENT ADDRS                   | IS LEARNER |
+------------------+---------+--------+---------------------------+-------------------------------------------------+------------+
| 65efecf6e9a81d9c | started | etcd-0 | http://10.211.55.100:2380 | http://10.211.55.100:2379,http://127.0.0.1:2379 |      false |
| b175d3aa415c26ed | started | etcd-1 | http://10.211.55.101:2380 | http://10.211.55.101:2379,http://127.0.0.1:2379 |      false |
| c1f2844c0614a5d1 | started | etcd-2 | http://10.211.55.102:2380 | http://10.211.55.102:2379,http://127.0.0.1:2379 |      false |
+------------------+---------+--------+---------------------------+-------------------------------------------------+------------+


# 查看集群状态
[root@node-2 ~]# etcdctl --endpoints=10.211.55.100:2379,10.211.55.101:2379,10.211.55.102:2379 --write-out=table endpoint status
+--------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|      ENDPOINT      |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+--------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| 10.211.55.100:2379 | 65efecf6e9a81d9c |  3.4.14 |   20 kB |      true |      false |        10 |         15 |                 15 |        |
| 10.211.55.101:2379 | b175d3aa415c26ed |  3.4.14 |   20 kB |     false |      false |        10 |         15 |                 15 |        |
| 10.211.55.102:2379 | c1f2844c0614a5d1 |  3.4.14 |   20 kB |     false |      false |        10 |         15 |                 15 |        |
+--------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```
### (7). 常用操作
```
# 增/改
[root@node-2 ~]# etcdctl --endpoints=10.211.55.100:2379,10.211.55.101:2379,10.211.55.102:2379 put foo "hello world"
OK

# 查
[root@node-2 ~]# etcdctl --endpoints=10.211.55.100:2379,10.211.55.101:2379,10.211.55.102:2379 get foo
foo
hello world

# 删
[root@node-2 ~]# etcdctl --endpoints=10.211.55.100:2379,10.211.55.101:2379,10.211.55.102:2379 del foo
1

# 校验,已经查询不到数据了
[root@node-2 ~]# etcdctl --endpoints=10.211.55.100:2379,10.211.55.101:2379,10.211.55.102:2379 get foo
```
