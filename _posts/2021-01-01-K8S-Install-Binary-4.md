---
layout: post
title: 'Kubernetes 二进制安装之Flannel安装与配置(四)'
date: 2021-01-01
author: 李新
tags: K8S  K8S安装
---

### (1). Kubernetes网络模型概念(CNI)
> Container Network Interface(CNI)是由Google和CoreOS主导研发.  
> Kubernetes网络模型设计基本要求:  
> 1. 一个Pod一个IP.  
> 2. 每个Pod独立IP,Pod内所有容器共享网络(同一个IP).  
> 3. 所有容器都可以与其它容器通信.  
> 4. 所有宿主机节点都可以与所有容器通信.  
> 5. CNI的实现者有: Flannel/Calico/Weaveworks/Open vSwitch/Contiv/Romana/Cilium   

### (2). CNI的底层实现
> CNI一般有两种实现:一种是基于隧道方案,另一种是基于路由表方案.

### (3). 基于隧道方案
> 1. 每一台宿主机上建立一个虚拟连接.  
> 2. 集群间的Docker容器,会通过虚拟链接(比如:flannel.1)进行报文传递,在传递过程中,会对现在的(TCP)数据报文,再次进行封包和解包.    
> 3. 优缺点:可以跨三层网络交互(宿主机之间可以不在同一网段).性能自然而然会有消耗.  

### (4). 基于路由表方案
> 1. 每一台宿主机上的Docker,都有一个独立的网段,并把网段信息和宿主机IP提交到Etcd中.
> 2. 宿主机实时watch ETCD中的配置,把Docker网段和对应的宿主机IP添加到路由表中.  
> 3. 宿主机,需要开启IPV4转发功能(echo "1" > /proc/sys/net/ipv4/ip_forward).   
> 4. 优缺点:通过路由表直接路由,不需要对(TCP)数据报文进行封包和解包,缺点:宿主机之间不能跨网段(只能在同一网段以内).

### (5). 写入需要分配的子网段到ETCD.

```
# 当前工作目录
[root@master ~]# pwd
/root

# 创建flannel临时工作目录
[root@master ~]# mkdir -p /root/k8s/flannel

# 进入flannel目录
[root@master ~]# cd k8s/flannel/
[root@master flannel]#

# https://coreos.com/flannel/docs/latest/configuration.html
# 创建flannel配置文件(为K8S划分一个大的网段)
[root@master flannel]# vi flannel-config.json
{
  "Network": "172.20.0.0/16",
  "SubnetMin": "172.20.10.0",
  "SubnetMax": "172.20.254.0",
  "Backend": {
    "Type":"vxlan"
  }
}

# 把定义的flannel配置往ETCD中进行注册.
[root@master flannel]# /opt/etcd/bin/etcdctl --ca-file=/opt/etcd/ssl/ca.pem --cert-file=/opt/etcd/ssl/server.pem --key-file=/opt/etcd/ssl/server-key.pem --endpoints="https://10.211.55.100:2379,https://10.211.55.101:2379,https://10.211.55.102:2379"  set  /coreos.com/network/config  <  flannel-config.json
{
  "Network": "172.20.0.0/16",
  "SubnetMin": "172.20.10.0",
  "SubnetMax": "172.20.254.0",
  "Backend": {
    "Type":"vxlan"
  }
}
```
### (6). 下载Flannel二进制包
> https://github.com/coreos/flannel/releases   
> 以下步骤都要在node-1/node-2上执行或配置.

```
# 创建k8s工作目录(config/bin/ssl)
$ mkdir -p /opt/kubernetes/{config,bin,ssl}

# 下载flannel
$ wget https://github.com/coreos/flannel/releases/download/v0.11.0/flannel-v0.11.0-linux-amd64.tar.gz

# 解压
$ tar -zxvf flannel-v0.11.0-linux-amd64.tar.gz

# 移动flannel到:/opt/kubernetes/bin目录下
$ mv flanneld mk-docker-opts.sh /opt/kubernetes/bin/

# 检查一下
$ ll /opt/kubernetes/bin/
-rwxr-xr-x 1 root root 35249016 Jan 29  2019 flanneld
-rwxr-xr-x 1 root root     2139 Oct 23  2018 mk-docker-opts.sh
```

### (7). 创建flanneld环境变量配置文件(/opt/kubernetes/config/flanneld)
> 以下步骤都要在node-1/node-2上执行或配置.

```
# 创建flannel环境变量配置文件
# 注意:环境变量的前后是有空格的,否则报错( invalid character " " in host name )
$ vi /opt/kubernetes/config/flanneld
FLANNEL_OPTIONS=" --etcd-endpoints=https://10.211.55.100:2379,https://10.211.55.101:2379,https://10.211.55.102:2379 \
--etcd-ca-file=/opt/etcd/ssl/ca.pem \
--etcd-cert-file=/opt/etcd/ssl/server.pem \
--etcd-key-file=/opt/etcd/ssl/server-key.pem "
```
### (8). 通过systemd来管理flannel(/usr/lib/systemd/system/flanneld.service)
> 以下步骤都要在node-1/node-2上执行或配置.

```
[Unit]
Description=Flanneld overlay address etcd agent
Documentation=https://github.com/coreos/flannel
After=network-online.target network.target
Before=docker.service

[Service]
Type=notify
EnvironmentFile=/opt/kubernetes/config/flanneld
ExecStart=/opt/kubernetes/bin/flanneld --ip-masq ${FLANNEL_OPTIONS}
# 启动之后执行命令(会为当前node节点生成子网)
ExecStartPost=/opt/kubernetes/bin/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/subnet.env
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
### (9). 配置docker(/usr/lib/systemd/system/docker.service)
> 以下内容要在node-1/node-2上执行

```
[Service]
# 1. 添加环境变量
EnvironmentFile=/run/flannel/subnet.env
# 2. 配置启动时的环境变量信息
ExecStart=/usr/bin/dockerd $DOCKER_NETWORK_OPTIONS
```
### (10). 启动flanneld,并重新启动docker
> 以下步骤都要在node-1/node-2上执行或配置.

```
# 加载配置
$ systemctl daemon-reload
# 开启系统运行自动加载
$ systemctl enable flanneld
# 启动flanneld
$ systemctl start flanneld
# 重启docker
$ systemctl restart docker
```

### (11). 检查下网络信息
> 查看docker0网卡是否与flannel网卡在同一网段.   

```
[root@node-1 ~]# ip addr|grep docker0
5: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    inet 172.20.97.1/24 brd 172.20.97.255 scope global docker0
[root@node-1 ~]# ip addr|grep flannel.1
6: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default
    inet 172.20.97.0/32 scope global flannel.1

	
[root@node-2 ~]# ip addr|grep docker0
5: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
    inet 172.20.28.1/24 brd 172.20.28.255 scope global docker0

[root@node-2 ~]# ip addr|grep flannel.1
6: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default
    inet 172.20.28.0/32 scope global flannel.1
```

### (12). 检查docker是否有应用上配置(--bip)
```
$ ps -ef|grep dockerd
root     24629     1  0 17:54 ?        00:00:01 /usr/bin/dockerd --bip=172.20.28.1/24 --ip-masq=false --mtu=1450
```
### (13). 验证容器互通
```
# 在node-1节点上创建容器
[root@node-1 ~]# docker run -d -it --rm --name linux_1 busybox
abc03a0fb52e8dc1ec739f13a662f3092ba939aebb37fc2fa061a140ad6c1027

# 进入node-1节点容器内部
[root@node-1 ~]# docker exec -it linux_1 sh
# 查看容器IP地址
/ # ip addr
7: eth0@if8: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1450 qdisc noqueue
    inet 172.20.97.2/24 brd 172.20.97.255 scope global eth0
	   
# ping node-2节点上的容器(linux_2的IP)
/ # ping 172.20.28.2
PING 172.20.28.2 (172.20.28.2): 56 data bytes
64 bytes from 172.20.28.2: seq=0 ttl=62 time=0.763 ms

# 在node-2节点上创建容器
[root@node-2 ~]# docker run -d -it --rm --name linux_2 busybox
d4ee6f5b7c20b3c76ea2d82b59275d47413c728c3fe0692fa459ff1c3ff11c60

# 进入node-2节点的容器内部
[root@node-2 ~]# docker exec -it linux_2 sh
# 查看IP地址
/ # ip addr
7: eth0@if8: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1450 qdisc noqueue
    inet 172.20.28.2/24 brd 172.20.28.255 scope global eth0

# ping node-1节点上的容器(linux_1的IP)
/ # ping 172.20.97.2
PING 172.20.97.2 (172.20.97.2): 56 data bytes
64 bytes from 172.20.97.2: seq=0 ttl=62 time=0.556 ms
64 bytes from 172.20.97.2: seq=1 ttl=62 time=0.474 ms
```

### (14). 查看ETCD数据
```
# 查看ETCD中子网数据
$ /opt/etcd/bin/etcdctl --ca-file=/opt/etcd/ssl/ca.pem --cert-file=/opt/etcd/ssl/server.pem --key-file=/opt/etcd/ssl/server-key.pem  ls  /coreos.com/network/subnets
/coreos.com/network/subnets/172.20.97.0-24
/coreos.com/network/subnets/172.20.28.0-24

# 查看subnets/172.20.97.0-24被哪台宿主机持有
$ /opt/etcd/bin/etcdctl --ca-file=/opt/etcd/ssl/ca.pem --cert-file=/opt/etcd/ssl/server.pem --key-file=/opt/etcd/ssl/server-key.pem  get  /coreos.com/network/subnets/172.20.97.0-24
{"PublicIP":"10.211.55.101","BackendType":"vxlan","BackendData":{"VtepMAC":"f6:df:b4:d0:5d:72"}}

# 查看subnets/172.20.28.0-24被哪台宿主机持有
$ /opt/etcd/bin/etcdctl --ca-file=/opt/etcd/ssl/ca.pem --cert-file=/opt/etcd/ssl/server.pem --key-file=/opt/etcd/ssl/server-key.pem  get  /coreos.com/network/subnets/172.20.28.0-24
{"PublicIP":"10.211.55.102","BackendType":"vxlan","BackendData":{"VtepMAC":"6e:eb:d6:21:ed:1e"}}
```