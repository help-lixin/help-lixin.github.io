---
layout: post
title: 'Kubernetes 二进制安装之集群准备(一)'
date: 2021-01-01
author: 李新
tags: K8S
---

### (1). 集群机器

|  IP            | 机器名称   |  部署组件      | 
|  ----          | ----     | ----           |
| 10.211.55.100  |  master  | kube-apiserver,kube-controller-manger,kube-scheduler,etcd |
| 10.211.55.101  |  node-1  | kubelet,kube-proxy,docker,etcd |
| 10.211.55.102  |  node-2  | kubelet,kube-proxy,docker,etcd |

### (2). 准备工作
```
# 所有机器关闭防火墙
$ systemctl stop firewalld
$ systemctl disable firewalld

# 所有机器关闭selinux
$ sed -i 's/enforcing/disabled/' /etc/selinux/config 
$ setenforce 0

# 所有机器关闭swap(编缉:/etc/fstab,注释掉最后一行:swap)
$ vi /etc/fstab
	# /dev/mapper/VolGroup-lv_swap swap                    swap    defaults        0 0

# 所有机器之间添加host与ip映射(/etc/hosts)
10.211.55.100   master
10.211.55.101   node-1
10.211.55.102   node-2

# 把IPV4流量转到到iptables里:
$ vi /etc/sysctl.d/k8s.conf
net.bridge.bridge‐nf‐call‐ip6tables = 1
net.bridge.bridge‐nf‐call‐iptables = 1

$ sysctl --system
```
### (3). 准备cfssl证书生成工具
```
[root@master ~]# curl -L https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 -o /usr/local/bin/cfssl
[root@master ~]# curl -L https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64  -o /usr/local/bin/cfssljson
[root@master ~]# curl -L https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64  -o /usr/local/bin/cfssl-certinfo
[root@master ~]# chmod u+x /usr/local/bin/cfssl*
```
### (4). 创建CA机构和CA机构证书(ca.pem/ca-key.pem)
```
# 当前工作目录
[root@master ~]# pwd
/root

# 1. 创建证书目录
[root@master ~]# mkdir -p ./k8s/{k8s-cert,etcd-cert}

# 2. 进入到etcd-cert目录
[root@master ~]# cd k8s/etcd-cert/
[root@master etcd-cert]# pwd
/root/k8s/etcd-cert

# 3. 创建用来生成CA文件的JSON配置文件
[root@master etcd-cert]# vi ca-config.json
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "www": {
         "expiry": "87600h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      }
    }
  }
}

# 4. 创建用来生成 CA证书签名请求(CSR)的JSON配置文件
[root@master etcd-cert]# vi ca-csr.json
{
    "CN": "etcd CA",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing"
        }
    ]
}

# 5. 生成CA证书（ca.pem）和密钥（ca-key.pem）
[root@master etcd-cert]# cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
2021/01/21 12:37:28 [INFO] generating a new CA key and certificate from CSR
2021/01/21 12:37:28 [INFO] generate received request
2021/01/21 12:37:28 [INFO] received CSR
2021/01/21 12:37:28 [INFO] generating key: rsa-2048
2021/01/21 12:37:29 [INFO] encoded CSR
2021/01/21 12:37:29 [INFO] signed certificate with serial number 126900092110139024664854647902926813080599228311

# 5.1 生成CA证书（ca.pem）和密钥（ca-key.pem）
[root@master etcd-cert]# ll
-rw-r--r-- 1 root root  289 Jan 21 12:35 ca-config.json   # 生成CA文件的JSON配置文件
-rw-r--r-- 1 root root  956 Jan 21 12:37 ca.csr           # CA证书请求文件
-rw-r--r-- 1 root root  209 Jan 21 12:36 ca-csr.json      # CA 证书签名请求(CSR)的JSON配置文件
-rw------- 1 root root 1675 Jan 21 12:37 ca-key.pem       # CA证书
-rw-r--r-- 1 root root 1265 Jan 21 12:37 ca.pem           # CA密钥
```
