---
layout: post
title: 'Kubernetes 二进制安装之结束篇'
date: 2021-01-01
author: 李新
tags: K8S K8S安装
---

### (1). K8S创建Pod时,是否会把机器名称(hostname)和IP地址保存在ETCD?
> 通过对ETCD的数据(Key)穷举,你会发现:   
> 呵呵呵,机器名称(hostname)和IP地址的映射在ETCD里压根就没有,那也就是说:K8S将这些数据保存在内存里.  

```
# 1. 查看pod
[root@master ~]# kubectl get pods
NAME                                  READY   STATUS    RESTARTS   AGE
busybox-deployment-74c549f577-rwl2w   1/1     Running   0          46m
nginx-6799fc88d8-mx7t6                1/1     Running   0          13m
nginx-6799fc88d8-n4t5j                1/1     Running   0          33m
nginx-6799fc88d8-shrn8                1/1     Running   0          13m

# 2. 查看key,只有一个是我们创建Flannel时创建的.
[root@master ~]# /opt/etcd/bin/etcdctl --ca-file=/opt/etcd/ssl/ca.pem --cert-file=/opt/etcd/ssl/server.pem --key-file=/opt/etcd/ssl/server-key.pem ls /
/coreos.com

# 2.1 查看:/coreos.com/network
# /coreos.com/network/config   是用来配置Flannel的网段.
$ /coreos.com/network/subnets  用来记录每个宿主机的分配的网段信息.
[root@master ~]# /opt/etcd/bin/etcdctl --ca-file=/opt/etcd/ssl/ca.pem --cert-file=/opt/etcd/ssl/server.pem --key-file=/opt/etcd/ssl/server-key.pem ls /coreos.com/network
/coreos.com/network/config
/coreos.com/network/subnets

# 2.2 我有两台宿主机,所以就有两个网段.
[root@master ~]# /opt/etcd/bin/etcdctl --ca-file=/opt/etcd/ssl/ca.pem --cert-file=/opt/etcd/ssl/server.pem --key-file=/opt/etcd/ssl/server-key.pem ls /coreos.com/network/subnets
/coreos.com/network/subnets/172.20.28.0-24
/coreos.com/network/subnets/172.20.97.0-24

# 2.3 查看:/coreos.com/network/subnets/172.20.28.0-24
[root@master ~]# /opt/etcd/bin/etcdctl --ca-file=/opt/etcd/ssl/ca.pem --cert-file=/opt/etcd/ssl/server.pem --key-file=/opt/etcd/ssl/server-key.pem get /coreos.com/network/subnets/172.20.28.0-24
{"PublicIP":"10.211.55.102","BackendType":"vxlan","BackendData":{"VtepMAC":"b6:83:de:ad:68:6d"}}

# 2.4 查看:/coreos.com/network/subnets/172.20.97.0-24
[root@master ~]# /opt/etcd/bin/etcdctl --ca-file=/opt/etcd/ssl/ca.pem --cert-file=/opt/etcd/ssl/server.pem --key-file=/opt/etcd/ssl/server-key.pem get /coreos.com/network/subnets/172.20.97.0-24
{"PublicIP":"10.211.55.101","BackendType":"vxlan","BackendData":{"VtepMAC":"fa:7f:96:a9:e8:bc"}}

# 2.5 查看:/coreos.com/network/config
[root@master ~]# /opt/etcd/bin/etcdctl --ca-file=/opt/etcd/ssl/ca.pem --cert-file=/opt/etcd/ssl/server.pem --key-file=/opt/etcd/ssl/server-key.pem get /coreos.com/network/config
{
  "Network": "172.20.0.0/16",
  "SubnetMin": "172.20.10.0",
  "SubnetMax": "172.20.254.0",
  "Backend": {
    "Type":"vxlan"
  }
}
```
### (2). 域名解析(Service底层实现)
> 在我搭建DNS解析时,我一直的理解是:我可以通过Pod的名称(hostname),来访问(而非IP)Pod.  
> 所以,造成在搭建DNS上花费了一个多小时后,到最后才理解到:  
> 在K8S创建Service的时候,实际也是建立DNS信息的时刻,换句话说:K8S的Service是建立在DNS之上的,它是对DNS解析的一种封装.从上面的分析我们也看得出来,DNS信息是保存在内存中的,而非ETCD中.    
> 也能理解K8S的这设计,因为:Pod的名称是动态的,求证过程如下:  

```
# 1. 查看pod信息
[root@master ~]# kubectl get pods,svc
NAME                                      READY   STATUS      RESTARTS   AGE
pod/busybox-deployment-74c549f577-rwl2w   0/1     Error       0          107m
pod/nginx-6799fc88d8-mx7t6                0/1     Completed   0          73m
pod/nginx-6799fc88d8-n4t5j                0/1     Completed   0          93m
pod/nginx-6799fc88d8-shrn8                0/1     Completed   0          73m

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)        AGE
service/kubernetes   ClusterIP   10.0.0.1     <none>        443/TCP        22h
service/nginx        NodePort    10.0.0.142   <none>        80:36826/TCP   92m

# 2. 进入容器内部(busybox-deployment-74c549f577-rwl2w)
[root@master ~]# kubectl exec -it busybox-deployment-74c549f577-rwl2w --  sh

# 2.1 尝试ping nginx pod的名称
/ # ping nginx-6799fc88d8-mx7t6
ping: bad address 'nginx-6799fc88d8-mx7t6'

# 2.1 尝试ping nginx pod的名称
/ # ping nginx-6799fc88d8-n4t5j
ping: bad address 'nginx-6799fc88d8-n4t5j'

# 2.1 尝试ping nginx pod的名称
/ # ping nginx-6799fc88d8-shrn8
ping: bad address 'nginx-6799fc88d8-shrn8'

# **************************************************************
# 2.2 尝试ping service的名称
# **************************************************************
/ # ping nginx
PING nginx (10.0.0.142): 56 data bytes
64 bytes from 10.0.0.142: seq=0 ttl=64 time=0.125 ms

# **************************************************************
# 2.3 获取得信息
# **************************************************************
/ # nslookup nginx
Server:         10.0.0.2
Address:        10.0.0.2:53

Non-authoritative answer:
Name:   nginx.default.svc.cluster.local
Address: 10.0.0.142
```
