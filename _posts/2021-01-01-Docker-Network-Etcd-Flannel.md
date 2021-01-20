---
layout: post
title: 'Docker 跨主机路由之:Etcd+Flannel'
date: 2021-01-01
author: 李新
tags: Docker
---

### (1).  环境准备

|  宿主IP         | 容器网段       | 主机名称     |
|  ----          | ----          |   ----      |
| 10.211.55.100  | 172.17.0.1/24 |  master     |
| 10.211.55.101  | 172.17.0.1/24 |  node-1     |
| 10.211.55.102  | 172.17.0.1/24 |  node-2     |

### (2). Etcd集群搭建
> ["Etcd集群搭建"](https://blog.lixin.help/2021/01/01/Etcd-Cluster.html)

```
# 查看集群成员列表
[root@master ~]# etcdctl  member list
65efecf6e9a81d9c: name=etcd-0 peerURLs=http://10.211.55.100:2380 clientURLs=http://10.211.55.100:2379,http://127.0.0.1:2379 isLeader=true
b175d3aa415c26ed: name=etcd-1 peerURLs=http://10.211.55.101:2380 clientURLs=http://10.211.55.101:2379,http://127.0.0.1:2379 isLeader=false
c1f2844c0614a5d1: name=etcd-2 peerURLs=http://10.211.55.102:2380 clientURLs=http://10.211.55.102:2379,http://127.0.0.1:2379 isLeader=false

# 查看集群状态
[root@master ~]# etcdctl cluster-health
member 65efecf6e9a81d9c is healthy: got healthy result from http://10.211.55.100:2379
member b175d3aa415c26ed is healthy: got healthy result from http://10.211.55.101:2379
member c1f2844c0614a5d1 is healthy: got healthy result from http://10.211.55.102:2379
cluster is healthy

# 查看是否有遗留数据
[root@master ~]# etcdctl ls /

# 清除遗留数据
# 可递归删除一个目录
# [root@master ~]# etcdctl rm -f /docker
```
### (3). 安装Flannel
```
# 三台机器都要安装flannel
$ yum  -y install  flannel
```
### (4). 配置Flannel配置文件
```
# flannel安装后,会创建一个配置文件(/etc/sysconfig/flanneld)
# 默认的ETCD URL是:http://127.0.0.1:2379
# 我这里不用配置,因为我本地有开启etcd
[root@node-2 system]# cat /etc/sysconfig/flanneld
	# Flanneld configuration options
	# etcd url location.  Point this to the server where etcd runs
	
	# ETCD的URL
	FLANNEL_ETCD_ENDPOINTS="http://127.0.0.1:2379"
	# etcd config key.  This is the configuration key that flannel queries
	# For address range assignment
	
	# Flannel在ETCD中的KEY前缀
	FLANNEL_ETCD_PREFIX="/atomic.io/network"
	
	# Any additional options that you want to pass
	#FLANNEL_OPTIONS=""
```
### (5). 在ETCD中创建Flannel需要的配置文件
> 以下内容是往etcd创建配置文件,所以,仅需要在任意一台宿主机上执行就可以了.   

```
# 创建flannel配置文件
[root@node-2 ~]# cat flannel-config.json
{
  "Network": "172.20.0.0/16",
  "SubnetMin": "172.20.10.0",
  "SubnetMax": "172.20.254.0",
  "Backend": {
    "Type":"vxlan"
  }
}

# 应用配置到etcd
[root@node-2 ~]# etcdctl set /atomic.io/network/config < flannel-config.json
{
  "Network": "172.20.0.0/16",
  "SubnetMin": "172.20.10.0",
  "SubnetMax": "172.20.254.0",
  "Backend": {
    "Type":"vxlan"
  }
}


# 检查配置文件是否已经成功
[root@node-2 ~]# etcdctl get /atomic.io/network/config
{
  "Network": "172.20.0.0/16",
  "SubnetMin": "172.20.10.0",
  "SubnetMax": "172.20.254.0",
  "Backend": {
    "Type":"vxlan"
  }
}
```
### (6). 启动Flannel
```
# 三台宿主机器都要执行
$ systemctl  start flanneld
```
### (7). 检查宿主机虚拟网卡(flannel.1)

```
# master
[root@master ~]# ip addr |grep flannel.1
13: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default
    inet 172.20.109.0/32 scope global flannel.1

# node-1
[root@node-1 ~]# ip addr |grep flannel.1
13: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default
    inet 172.20.106.0/32 scope global flannel.1

# node-2
[root@node-2 ~]# ip addr |grep flannel.1
13: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default
    inet 172.20.11.0/32 scope global flannel.1
```
### (8). 检查宿主机上:/run/flannel/subnet.env
```
# master
[root@master ~]# cat /run/flannel/subnet.env
FLANNEL_NETWORK=172.20.0.0/16
FLANNEL_SUBNET=172.20.109.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=false

# node-1
[root@node-1 ~]# cat /run/flannel/subnet.env
FLANNEL_NETWORK=172.20.0.0/16
FLANNEL_SUBNET=172.20.106.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=false

# node-2
[root@node-2 ~]# cat /run/flannel/subnet.env
FLANNEL_NETWORK=172.20.0.0/16
FLANNEL_SUBNET=172.20.11.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=false
```
### (9). 配置Docker环境变量(DOCKER_OPTS)
```
# 三台机器要都执行
# 该命令会在宿主机上创建:/run/docker_opts.env
$ /usr/libexec/flannel/mk-docker-opts.sh -c


# /run/docker_opts.env 实际就是配置DOCKER_OPTS
# bip会改变宿主机上docker0的网卡信息
# [root@node-2 ~]# cat /run/docker_opts.env
# DOCKER_OPTS=" --bip=172.20.11.1/24 --ip-masq=true --mtu=1450"
```
### (10). 配置docker启动服务(/lib/systemd/system/docker.service)
> 三台机器都要进行配置,这一步:是让上一步生成的环境变量与docker绑定.  

```
$ vi  /lib/systemd/system/docker.service
# 1. 在ExecStart变量前面,添加环境变量文件
EnvironmentFile=/run/docker_opts.env
# 2. 在启动命令中添加环境变量
ExecStart=/usr/bin/dockerd  $DOCKER_OPTS 
```
### (11). 启动docker
```
$ systemctl daemon-reload
$ systemctl restart docker
```
### (12). 检查docker0网卡信息
> 检查下docker0是否应用了环境变量的配置(/run/docker_opts.env)  
> 即IP按照环境变量的配置生效了.   

```
# master
[root@master ~]# ip addr
5: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:19:fa:e4:7a brd ff:ff:ff:ff:ff:ff
    inet 172.20.109.1/24 brd 172.20.109.255 scope global docker0
       valid_lft forever preferred_lft forever
13: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default
    link/ether 12:35:e1:43:87:af brd ff:ff:ff:ff:ff:ff
    inet 172.20.109.0/32 scope global flannel.1
	
# nod-1
[root@node-1 ~]# ip addr
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:09:b2:6c:1c brd ff:ff:ff:ff:ff:ff
    inet 172.20.106.1/24 brd 172.20.106.255 scope global docker0
       valid_lft forever preferred_lft forever
13: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default
    link/ether 06:c2:9a:d0:10:52 brd ff:ff:ff:ff:ff:ff
    inet 172.20.106.0/32 scope global flannel.1

# node-2
[root@node-2 ~]# ip addr
5: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:45:c8:77:a8 brd ff:ff:ff:ff:ff:ff
    inet 172.20.11.1/24 brd 172.20.11.255 scope global docker0
13: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default
    link/ether 16:89:d9:b5:af:b5 brd ff:ff:ff:ff:ff:ff
    inet 172.20.11.0/32 scope global flannel.1
```
### (13). 创建容器
```
# master节点创建容器
[root@master ~]# docker run -d -it  --rm  --name linux_1 busybox:latest
3b99cc32b39a46f9ce9cc6dffc41343b002610c8f3dd8256900d566d74310826

# node-1节点创建容器
[root@node-1 ~]# docker run -d -it  --rm  --name linux_2 busybox:latest
d48800d863caddbf3d8ed53399ee1f21e27d3b7580408df5238ff829dda0a71d

# node-2节点创建容器
[root@node-2 ~]# docker run -d -it  --rm  --name linux_3 busybox:latest
e26d5aa87cda354b8af5c3a66ee4591b978a9282f6ba5853bd24a66199a20a80
```

### (14). 进入容器内部查看容器IP
```
# 在master节hok进入容器内部
[root@master ~]# docker exec -it linux_1 sh
# 查看容器内部的IP地址
/ # ip addr
7: eth0@if8: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1450 qdisc noqueue
    link/ether 02:42:ac:14:11:02 brd ff:ff:ff:ff:ff:ff
    inet 172.20.17.2/24 brd 172.20.17.255 scope global eth0
       valid_lft forever preferred_lft forever

# 在node-1节hok进入容器内部
[root@node-1 ~]# docker exec -it linux_2 sh
# 查看容器内部的IP地址
/ # ip addr
7: eth0@if8: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1450 qdisc noqueue
    link/ether 02:42:ac:14:60:02 brd ff:ff:ff:ff:ff:ff
    inet 172.20.96.2/24 brd 172.20.96.255 scope global eth0
       valid_lft forever preferred_lft forever

# 在node-2节hok进入容器内部
[root@node-2 ~]# docker exec -it linux_3 sh
# 查看容器内部的IP地址
/ # ip addr
7: eth0@if8: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1450 qdisc noqueue
    link/ether 02:42:ac:14:13:02 brd ff:ff:ff:ff:ff:ff
    inet 172.20.19.2/24 brd 172.20.19.255 scope global eth0
       valid_lft forever preferred_lft forever
```

### (15). 容器内部互相ping
```
# 在master节点的linux_1容器内部测试,ping其它服务器
# linux_2
/ # ping 172.20.96.2
PING 172.20.96.2 (172.20.96.2): 56 data bytes
64 bytes from 172.20.96.2: seq=0 ttl=62 time=0.618 ms

# linux_3
/ # ping 172.20.19.2
PING 172.20.19.2 (172.20.19.2): 56 data bytes
64 bytes from 172.20.19.2: seq=2 ttl=62 time=0.534 ms

# 测试ping域名,发现不能访问.
# 应该要自建DNS服务器了
/ # ping linux_1
ping: bad address 'linux_1'
/ # ping linux_2
ping: bad address 'linux_2'
/ # ping linux_3
ping: bad address 'linux_3'
```