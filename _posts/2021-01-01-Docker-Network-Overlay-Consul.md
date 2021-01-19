---
layout: post
title: 'Docker 跨主机路由之Overlay(Consul)'
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

### (2). Consul集群搭建(略)
> ["Consul集群搭建"](https://blog.lixin.help/2021/01/01/Consul-Cluster.html)

### (3). 所有宿主机配置daemon.json
```

# master
[root@master ~]# cat /etc/docker/daemon.json
{
    "cluster-store" : "consul://10.211.55.100:8500",
    "cluster-advertise" : "10.211.55.100:2376"
}


# node-1
[root@node-1 ~]# cat /etc/docker/daemon.json
{
    "cluster-store" : "consul://10.211.55.101:8500",
    "cluster-advertise" : "10.211.55.101:2376"
}

# node-2
[root@node-2 ~]# cat  /etc/docker/daemon.json
{
    "cluster-store" : "consul://10.211.55.102:8500",
    "cluster-advertise" : "10.211.55.102:2376"
}
```
### (4). 所有宿主机重启docker
```
$ systemctl daemon-reload
$ systemctl restart docker
```
### (5). 创建Docker网络
> 在任意一台机器上创建网络,它们之间会同步网络配置信息.

```
# 创网络名称(consul-net)
[root@node-2 ~]# docker network create --driver overlay consul-net
03a05ac31d22a08f3a6aea72f771311913b268600fb09e0fbe46d7d3538aadf0

# 会自动同步创建的网络信息
# node-2查看网络
[root@node-2 ~]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
5e17791face0        bridge              bridge              local
03a05ac31d22        consul-net          overlay             global
fdbb0ca2b22c        host                host                local
1f23aa33de6d        none                null                local

# master查看网络
[root@master ~]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
010a938724b6        bridge              bridge              local
03a05ac31d22        consul-net          overlay             global
aa3f0e62d025        host                host                local
2c8522abc360        none                null                local
```
### (6). 创建容器
```
# 创建容器(linux_1)
# 注意:指定了--network
[root@master ~]# docker run -d -it --network consul-net  --rm  --name linux_1 busybox:latest
029fb867a0e5dad750371d28aa7203f663377ceb5ec41da164cdb1fa7085c085
[root@master ~]# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
029fb867a0e5        busybox:latest      "sh"                50 seconds ago      Up 49 seconds                           linux_1


# 创建容器(linux_2)
# 注意:指定了--network
[root@node-1 ~]# docker run -d -it --network consul-net  --rm  --name linux_2 busybox:latest
f76ce8f24286725fe6a1c62fb5374adc1eb35006c0f67f41223bd1739bf696c0
[root@node-1 ~]# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
f76ce8f24286        busybox:latest      "sh"                31 seconds ago      Up 30 seconds                           linux_2

# 创建容器(linux_3)
# 注意:指定了--network
[root@node-2 ~]# docker run -d -it --network consul-net  --rm  --name linux_3 busybox:latest
0da77b574a5afd2004b4f0752f8f07e77418d5c50074de33dc7c10bd1bd44c9c
[root@node-2 ~]# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
0da77b574a5a        busybox:latest      "sh"                10 seconds ago      Up 9 seconds                            linux_3
```
### (7). 进入容器内部查看IP信息
> 容器的IP是由consul统一分派.

```
# 在master宿主机上,进入容器(linux_1)
[root@master ~]# docker exec -it linux_1 sh

# 查看容器内部IP地址
/ # ip addr
10: eth0@if11: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1450 qdisc noqueue
    link/ether 02:42:0a:00:00:02 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.2/24 brd 10.0.0.255 scope global eth0
       valid_lft forever preferred_lft forever
13: eth1@if14: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue
    link/ether 02:42:ac:12:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.2/16 brd 172.18.255.255 scope global eth1
       valid_lft forever preferred_lft forever
	   
# 在node-1节点进入容器(linux_2)	   
[root@node-1 ~]# docker exec -it linux_2 sh
# 查看容器内部IP地址
/ # ip addr
10: eth0@if11: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1450 qdisc noqueue
    link/ether 02:42:0a:00:00:03 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.3/24 brd 10.0.0.255 scope global eth0
       valid_lft forever preferred_lft forever
13: eth1@if14: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue
    link/ether 02:42:ac:12:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.2/16 brd 172.18.255.255 scope global eth1
       valid_lft forever preferred_lft forever


# 在node-2节点进入容器(linux_3)
[root@node-2 ~]# docker exec -it linux_3 sh
# 查看容器内部IP地址
/ # ip addr
10: eth0@if11: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1450 qdisc noqueue
    link/ether 02:42:0a:00:00:04 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.4/24 brd 10.0.0.255 scope global eth0
       valid_lft forever preferred_lft forever
13: eth1@if14: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue
    link/ether 02:42:ac:12:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.2/16 brd 172.18.255.255 scope global eth1
       valid_lft forever preferred_lft forever
```
### (8). 在容器内部进入测试连接
> 测试ping域名,看能否ping通. 

```
# 在宿主机(master)上进入容器内部
[root@master ~]# docker exec -it linux_1 sh
# 在容器内部,ping域名(linux_2)
/ # ping linux_2
PING linux_2 (10.0.0.3): 56 data bytes
64 bytes from 10.0.0.3: seq=0 ttl=64 time=0.747 ms

# 在容器内部,ping域名(linux_3)
/ # ping linux_3
PING linux_3 (10.0.0.4): 56 data bytes
64 bytes from 10.0.0.4: seq=0 ttl=64 time=0.802 ms
```
