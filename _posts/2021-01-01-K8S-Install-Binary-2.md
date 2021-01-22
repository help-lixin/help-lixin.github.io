---
layout: post
title: 'Kubernetes 二进制安装之Etcd集群(二)'
date: 2021-01-01
author: 李新
tags: K8S  K8S安装
---

### (1). 使用自签CA签发Etcd HTTPS证书
```
# 1. 创建etcd证书请求文件
[root@master etcd-cert]# vi server-csr.json
{
    "CN": "etcd",
    "hosts": [
    "10.211.55.100",
    "10.211.55.101",
    "10.211.55.102"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "BeiJing",
            "ST": "BeiJing"
        }
    ]
}


# 2. 创建etcd证书(ca.pem/cak-key.pem/ca-config.pem)
# cfssljson -bare server 指定证书生成的文件名为(server)
[root@master etcd-cert]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www server-csr.json | cfssljson -bare server

# 3. 查看生成的文件
[root@master etcd-cert]# ll
-rw-r--r-- 1 root root  289 Jan 21 12:35 ca-config.json
-rw-r--r-- 1 root root  956 Jan 21 12:37 ca.csr
-rw-r--r-- 1 root root  209 Jan 21 12:36 ca-csr.json
-rw------- 1 root root 1675 Jan 21 12:37 ca-key.pem
-rw-r--r-- 1 root root 1265 Jan 21 12:37 ca.pem

-rw-r--r-- 1 root root 1013 Jan 21 13:07 server.csr         # 证书请求文件
-rw-r--r-- 1 root root  290 Jan 21 13:05 server-csr.json    # 证书生成请求JSON
-rw------- 1 root root 1675 Jan 21 13:07 server-key.pem     # 私钥
-rw-r--r-- 1 root root 1342 Jan 21 13:07 server.pem         # 公钥
```

### (2). 部署Etcd
```
# 当前工作目录
[root@master ~]# pwd
/root

# 1.创建etcd应用程序目录
[root@master ~]# mkdir -p  /opt/etcd/{bin,config,data,ssl}

# 2. 下载并部署etcd
[root@master ~]# wget https://github.com/etcd-io/etcd/releases/download/v3.3.11/etcd-v3.3.11-linux-amd64.tar.gz
# 解压
[root@master ~]# tar -zxvf etcd-v3.3.11-linux-amd64.tar.gz
# 移动(etcd/etcdctl)到bin目录
[root@master ~]# mv etcd-v3.3.11-linux-amd64/etcd* /opt/etcd/bin/
```
### (3). 创建etcd环境变量配置文件(/opt/etcd/config/etcd.conf)
```
# 创建etcd.conf
[root@master ~]# vi /opt/etcd/config/etcd.conf
#[Member]
ETCD_NAME="etcd-0"
ETCD_DATA_DIR="/opt/etcd/data"
ETCD_LISTEN_PEER_URLS="https://10.211.55.100:2380"
ETCD_LISTEN_CLIENT_URLS="https://10.211.55.100:2379"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://10.211.55.100:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://10.211.55.100:2379"
ETCD_INITIAL_CLUSTER="etcd-0=https://10.211.55.100:2380,etcd-1=https://10.211.55.101:2380,etcd-2=https://10.211.55.102:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
```
### (4). 通过systemd来管理etcd
```
[root@master ~]# vi /usr/lib/systemd/system/etcd.service
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
EnvironmentFile=/opt/etcd/config/etcd.conf
ExecStart=/opt/etcd/bin/etcd \
--name=${ETCD_NAME} \
--data-dir=${ETCD_DATA_DIR} \
--listen-peer-urls=${ETCD_LISTEN_PEER_URLS} \
--listen-client-urls=${ETCD_LISTEN_CLIENT_URLS},http://127.0.0.1:2379 \
--advertise-client-urls=${ETCD_ADVERTISE_CLIENT_URLS} \
--initial-advertise-peer-urls=${ETCD_INITIAL_ADVERTISE_PEER_URLS} \
--initial-cluster ${ETCD_INITIAL_CLUSTER} \
--initial-cluster-token=${ETCD_INITIAL_CLUSTER_TOKEN} \
--initial-cluster-state=${ETCD_INITIAL_CLUSTER_STATE} \
--cert-file=/opt/etcd/ssl/server.pem \
--key-file=/opt/etcd/ssl/server-key.pem \
--peer-cert-file=/opt/etcd/ssl/server.pem \
--peer-key-file=/opt/etcd/ssl/server-key.pem \
--trusted-ca-file=/opt/etcd/ssl/ca.pem \
--peer-trusted-ca-file=/opt/etcd/ssl/ca.pem
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```
### (5). 拷贝证书到etcd目录
```
# 拷贝ca证书
[root@master ~]# cp ./k8s/etcd-cert/ca.pem  /opt/etcd/ssl/

# 拷贝etcd证书到ssl
[root@master ~]# cp ./k8s/etcd-cert/server*.pem   /opt/etcd/ssl/

# 查看ssl证书是否拷贝到位.
[root@master ~]# ll /opt/etcd/ssl/
-rw-r--r-- 1 root root 1265 Jan 21 14:01 ca.pem
-rw------- 1 root root 1675 Jan 21 14:00 server-key.pem
-rw-r--r-- 1 root root 1342 Jan 21 14:00 server.pem
```
### (6). 分发文件到其它节点
```
# etcd目录
[root@master ~]# scp -r /opt/etcd/ root@10.211.55.101:/opt/
# /usr/lib/systemd/system/etcd.service
[root@master ~]# scp /usr/lib/systemd/system/etcd.service root@10.211.55.101:/usr/lib/systemd/system/etcd.service

# etcd目录
[root@master ~]# scp -r /opt/etcd/ root@10.211.55.102:/opt/
# /usr/lib/systemd/system/etcd.service
[root@master ~]# scp /usr/lib/systemd/system/etcd.service root@10.211.55.102:/usr/lib/systemd/system/etcd.service
```
### (7). 修改其它两个节点的配置文件(/opt/etcd/config/etcd.conf)

> node-1(/opt/etcd/config/etcd.conf)

```
ETCD_NAME="etcd-1"
ETCD_DATA_DIR="/opt/etcd/data"
ETCD_LISTEN_PEER_URLS="https://10.211.55.101:2380"
ETCD_LISTEN_CLIENT_URLS="https://10.211.55.101:2379"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://10.211.55.101:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://10.211.55.101:2379"
ETCD_INITIAL_CLUSTER="etcd-0=https://10.211.55.100:2380,etcd-1=https://10.211.55.101:2380,etcd-2=https://10.211.55.102:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"

# new是新集群,existing表示加入已有集群
ETCD_INITIAL_CLUSTER_STATE="new"
```

> node-2(/opt/etcd/config/etcd.conf)

```
ETCD_NAME="etcd-2"
ETCD_DATA_DIR="/opt/etcd/data"
ETCD_LISTEN_PEER_URLS="https://10.211.55.102:2380"
ETCD_LISTEN_CLIENT_URLS="https://10.211.55.102:2379"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://10.211.55.102:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://10.211.55.102:2379"
ETCD_INITIAL_CLUSTER="etcd-0=https://10.211.55.100:2380,etcd-1=https://10.211.55.101:2380,etcd-2=https://10.211.55.102:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"

# new是新集群,existing表示加入已有集群
ETCD_INITIAL_CLUSTER_STATE="new"
```

### (8). 所有节点启动etcd
> 所有节点都要做这件事情

```
# 加载配置
$ systemctl daemon-reload
# 开启系统运行自动加载
$ systemctl enable etcd
# 启动etcd
$ systemctl start etcd
```
### (10). 查看日志
```
# 查看日志
$ tail -500f /var/log/messages
```
### (11). 查看集群健康状态
```
# 指定证书并查看集群状态
[root@node-2 ~]# /opt/etcd/bin/etcdctl --ca-file=/opt/etcd/ssl/ca.pem --cert-file=/opt/etcd/ssl/server.pem --key-file=/opt/etcd/ssl/server-key.pem cluster-health
member 548fd5befc7ba852 is healthy: got healthy result from https://10.211.55.101:2379
member bcbc345cc52a96e9 is healthy: got healthy result from https://10.211.55.100:2379
member bd3d2a746b4da884 is healthy: got healthy result from https://10.211.55.102:2379
cluster is healthy

# 指定证书和endpoints,并查看集群状态
[root@master ~]# /opt/etcd/bin/etcdctl --ca-file=/opt/etcd/ssl/ca.pem --cert-file=/opt/etcd/ssl/server.pem --key-file=/opt/etcd/ssl/server-key.pem --endpoints="https://10.211.55.100:2379,https://10.211.55.101:2379,https://10.211.55.102:2379"  cluster-health
member 548fd5befc7ba852 is healthy: got healthy result from https://10.211.55.101:2379
member bcbc345cc52a96e9 is healthy: got healthy result from https://10.211.55.100:2379
member bd3d2a746b4da884 is healthy: got healthy result from https://10.211.55.102:2379
cluster is healthy

# 不指定证书查看集群状态
[root@node-2 ~]# /opt/etcd/bin/etcdctl cluster-health
failed to check the health of member 548fd5befc7ba852 on https://10.211.55.101:2379: Get https://10.211.55.101:2379/health: x509: certificate signed by unknown authority
member 548fd5befc7ba852 is unreachable: [https://10.211.55.101:2379] are all unreachable

failed to check the health of member bcbc345cc52a96e9 on https://10.211.55.100:2379: Get https://10.211.55.100:2379/health: x509: certificate signed by unknown authority
member bcbc345cc52a96e9 is unreachable: [https://10.211.55.100:2379] are all unreachable

failed to check the health of member bd3d2a746b4da884 on https://10.211.55.102:2379: Get https://10.211.55.102:2379/health: x509: certificate signed by unknown authority
member bd3d2a746b4da884 is unreachable: [https://10.211.55.102:2379] are all unreachable
cluster is unavailable
```