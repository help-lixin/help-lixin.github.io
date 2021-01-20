---
layout: post
title: 'Docker 跨主机路由之:Etcd+Flannel(host-gw)'
date: 2021-01-01
author: 李新
tags: Docker
---

### (0).  环境准备

|  宿主IP         | 容器网段       | 主机名称     |
|  ----          | ----          |   ----      |
| 10.211.55.100  | 172.17.0.1/24 |  master     |
| 10.211.55.101  | 172.17.0.1/24 |  node-1     |
| 10.211.55.102  | 172.17.0.1/24 |  node-2     |

### (1).  前期准备工作 
```
# 所有机器关闭防火墙
$ systemctl stop firewalld
$ systemctl disable firewalld

# 所有机器关闭selinux
$ sed -i 's/enforcing/disabled/' /etc/selinux/config 
$ setenforce 0

# 开启数据包转发功能
$ echo "1" > /proc/sys/net/ipv4/ip_forward
```

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
    "Type":"host-gw"
  }
}

# 应用配置到etcd
[root@node-2 ~]# etcdctl set /atomic.io/network/config < flannel-config.json
{
  "Network": "172.20.0.0/16",
  "SubnetMin": "172.20.10.0",
  "SubnetMax": "172.20.254.0",
  "Backend": {
    "Type":"host-gw"
  }
}


# 检查配置文件是否已经成功
[root@node-2 ~]# etcdctl get /atomic.io/network/config
{
  "Network": "172.20.0.0/16",
  "SubnetMin": "172.20.10.0",
  "SubnetMax": "172.20.254.0",
  "Backend": {
    "Type":"host-gw"
  }
}
```
### (6). 启动Flannel
```
# 三台宿主机器都要执行
$ systemctl  start flanneld
```
### (7). 检查宿主机上:/run/flannel/subnet.env
```
# master
[root@master ~]# cat /run/flannel/subnet.env
FLANNEL_NETWORK=172.20.0.0/16
FLANNEL_SUBNET=172.20.91.1/24
FLANNEL_MTU=1500
FLANNEL_IPMASQ=false

# node-1
[root@node-1 ~]# cat /run/flannel/subnet.env
FLANNEL_NETWORK=172.20.0.0/16
FLANNEL_SUBNET=172.20.101.1/24
FLANNEL_MTU=1500
FLANNEL_IPMASQ=false

#node-2
[root@node-2 ~]# cat /run/flannel/subnet.env
FLANNEL_NETWORK=172.20.0.0/16
FLANNEL_SUBNET=172.20.47.1/24
FLANNEL_MTU=1500
FLANNEL_IPMASQ=false
```
### (8). 配置Docker环境变量(DOCKER_OPTS)
```
# 三台机器要都执行
# 该命令会在宿主机上创建:/run/docker_opts.env
$ /usr/libexec/flannel/mk-docker-opts.sh -c


# /run/docker_opts.env 实际就是配置DOCKER_OPTS
# bip会改变宿主机上docker0的网卡信息
# [root@node-2 ~]# cat /run/docker_opts.env
# DOCKER_OPTS=" --bip=172.20.11.1/24 --ip-masq=true --mtu=1450"
```
### (9). 配置docker启动服务(/lib/systemd/system/docker.service)
> 三台机器都要进行配置,这一步:是让上一步生成的环境变量与docker绑定.  

```
$ vi  /lib/systemd/system/docker.service
# 1. 在ExecStart变量前面,添加环境变量文件
EnvironmentFile=/run/docker_opts.env
# 2. 在启动命令中添加环境变量
ExecStart=/usr/bin/dockerd  $DOCKER_OPTS 
```
### (10). 重启docker
```
$ systemctl daemon-reload
$ systemctl restart docker
```
### (11). 检查docker0网卡信息
> 检查下docker0是否应用了环境变量的配置(/run/docker_opts.env)  
> 即IP按照环境变量的配置生效了.   

```
# master
[root@master ~]#  ip addr|grep docker0
5: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    inet 172.20.91.1/24 brd 172.20.91.255 scope global docker0

# node-1
[root@node-1 ~]#  ip addr|grep docker0
5: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    inet 172.20.101.1/24 brd 172.20.101.255 scope global docker0

# node-2
[root@node-2 ~]#  ip addr|grep docker0
5: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    inet 172.20.47.1/24 brd 172.20.47.255 scope global docker0
	
```
### (12). 检查路由信息
> host-gw模式实际就是添加了静态路由

```
[root@node-2 ~]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.211.55.1     0.0.0.0         UG    100    0        0 eth0
10.211.55.0     0.0.0.0         255.255.255.0   U     100    0        0 eth0
172.20.47.0     0.0.0.0         255.255.255.0   U     0      0        0 docker0
172.20.91.0     10.211.55.100   255.255.255.0   UG    0      0        0 eth0
172.20.101.0    10.211.55.101   255.255.255.0   UG    0      0        0 eth0
192.168.122.0   0.0.0.0         255.255.255.0   U     0      0        0 virbr0
```
### (13). 创建容器
```
# master节点创建容器
[root@master ~]# docker run -d -it  --rm  --name linux_1 busybox:latest
0e078e602afc2a1f7382bde7b084db1e0c8b5da26a7d329e46bdb659fdb252cf

# node-1节点创建容器
[root@node-1 ~]# docker run -d -it  --rm  --name linux_2 busybox:latest
5be9c915b8df84f96155bfa288b42586f153b75e14d86e57525582696a7009c0

# node-2节点创建容器
[root@node-2 ~]# docker run -d -it  --rm  --name linux_3 busybox:latest
047d90fc63108123afdec7370d28627dd10902b1eba73ea322ed58c20c6828a7
```
### (14). 进入容器内部查看IP
```
# master节点进入容器内部
[root@master ~]# docker exec -it linux_1 sh
# 查看容器IP
/ # ip addr
8: eth0@if9: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue
    inet 172.20.91.2/24 brd 172.20.91.255 scope global eth0

# node-1节点进入容器内部
[root@node-1 ~]# docker exec -it linux_2 sh
# 查看容器IP
/ # ip addr
8: eth0@if9: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue
    inet 172.20.101.2/24 brd 172.20.101.255 scope global eth0

# node-2节点进入容器内部
[root@node-2 ~]# docker exec -it linux_3 sh
# 查看容器IP
/ # ip addr
6: eth0@if7: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue
    inet 172.20.47.2/24 brd 172.20.47.255 scope global eth0
```
### (15). 测试能否ping通
```
# 进入master节点的容器内部
[root@master ~]# docker exec -it linux_1 sh
# 查看IP
/ # ip addr
8: eth0@if9: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue
    link/ether 02:42:ac:14:5b:02 brd ff:ff:ff:ff:ff:ff
    inet 172.20.91.2/24 brd 172.20.91.255 scope global eth0
       valid_lft forever preferred_lft forever
	   
# 测试linux_2
/ # ping 172.20.101.2
PING 172.20.101.2 (172.20.101.2): 56 data bytes
64 bytes from 172.20.101.2: seq=0 ttl=62 time=0.500 ms

# 测试linux_3
/ # ping 172.20.47.2
PING 172.20.47.2 (172.20.47.2): 56 data bytes
64 bytes from 172.20.47.2: seq=0 ttl=62 time=0.548 ms


# 进入node-1的容器节点
[root@node-1 ~]# docker exec -it linux_2 sh
# 查看IP
/ # ip addr
8: eth0@if9: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue
    inet 172.20.101.2/24 brd 172.20.101.255 scope global eth0

# 测试linux_1
/ # ping 172.20.91.2
PING 172.20.91.2 (172.20.91.2): 56 data bytes
64 bytes from 172.20.91.2: seq=0 ttl=62 time=0.486 ms

# 测试linux_3
/ # ping 172.20.47.2
PING 172.20.47.2 (172.20.47.2): 56 data bytes
64 bytes from 172.20.47.2: seq=0 ttl=62 time=0.956 ms


# 进入node-2节点的容器内部
[root@node-2 ~]# docker exec -it linux_3 sh
# 查看容器IP地址
/ # ipaddr
6: eth0@if7: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue
    inet 172.20.47.2/24 brd 172.20.47.255 scope global eth0

# 测试linux_1
/ # ping 172.20.91.2
PING 172.20.91.2 (172.20.91.2): 56 data bytes
64 bytes from 172.20.91.2: seq=0 ttl=62 time=0.499 ms

# 测试linux_2
/ # ping 172.20.101.2
PING 172.20.101.2 (172.20.101.2): 56 data bytes
64 bytes from 172.20.101.2: seq=0 ttl=62 time=0.574 ms
```
