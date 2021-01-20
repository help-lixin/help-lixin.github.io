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
$ wget https://github.com/etcd-io/etcd/releases/download/v3.3.11/etcd-v3.3.11-linux-amd64.tar.gz

# 解压
$ tar -xvf etcd-v3.3.11-linux-amd64.tar.gz

# 移动etcd*可执行文件到PATH目录下
$ mv etcd-v3.3.11-linux-amd64/etcd* /usr/local/bin/


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
[root@master ~]# nohup etcd --config-file=/etc/etcd/etcd-cfg.yml & 
[root@node-1 ~]# nohup etcd --config-file=/etc/etcd/etcd-cfg.yml &
[root@node-2 ~]# nohup etcd --config-file=/etc/etcd/etcd-cfg.yml & 
```
### (6). 查看集群成员
```
# 查看集群成员列表
[root@master ~]# etcdctl member list
65efecf6e9a81d9c: name=etcd-0 peerURLs=http://10.211.55.100:2380 clientURLs=http://10.211.55.100:2379,http://127.0.0.1:2379 isLeader=true
b175d3aa415c26ed: name=etcd-1 peerURLs=http://10.211.55.101:2380 clientURLs=http://10.211.55.101:2379,http://127.0.0.1:2379 isLeader=false
c1f2844c0614a5d1: name=etcd-2 peerURLs=http://10.211.55.102:2380 clientURLs=http://10.211.55.102:2379,http://127.0.0.1:2379 isLeader=false

# 查看集群状态
[root@master ~]# etcdctl cluster-health
member 65efecf6e9a81d9c is healthy: got healthy result from http://10.211.55.100:2379
member b175d3aa415c26ed is healthy: got healthy result from http://10.211.55.101:2379
member c1f2844c0614a5d1 is healthy: got healthy result from http://10.211.55.102:2379
cluster is healthy
```
