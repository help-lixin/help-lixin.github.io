---
layout: post
title: 'Docker 跨主机路由之静态路由'
date: 2021-01-01
author: 李新
tags: Docker
---

### (1). 环境准备

|  宿主IP         | 容器网段         | 主机名称     |
|  ----          | ----            |   ----      |
| 10.211.55.100  | 172.17.1.254/24 |  master     |
| 10.211.55.101  | 172.17.2.254/24 |  node-1     |

### (2). 修改master节点docker0的IP地址
```
[root@master ~]# cd /etc/docker/
# 添加daemon.json配置网址
[root@master docker]# cat daemon.json
{
  "bip":"172.17.1.254/24"
}

# 重新加载配置
[root@master docker]# systemctl daemon-reload
# 重启docker
[root@master docker]# systemctl restart docker

# 检查docker0的地址
[root@master docker]# ip addr
5: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:5d:b2:9a:d5 brd ff:ff:ff:ff:ff:ff
    inet 172.17.1.254/24 brd 172.17.1.255 scope global docker0
       valid_lft forever preferred_lft forever
```
### (3). 修改node-1节点docker0的IP地址
```
[root@node-1 ~]# cd /etc/docker/

# 添加daemon.json配置网址
[root@master docker]# cat daemon.json
{
  "bip":"172.17.2.254/24"
}

# 重新加载配置
[root@master docker]# systemctl daemon-reload
# 重启docker
[root@master docker]# systemctl restart docker

# 检查docker0的地址
[root@node-1 docker]# ip addr
5: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:1f:04:2b:f9 brd ff:ff:ff:ff:ff:ff
    inet 172.17.2.254/24 brd 172.17.2.255 scope global docker0
       valid_lft forever preferred_lft forever
```
### (4). 在master节点添加路由
```
# 通往:172.17.2.0的地址,网关为:10.211.55.101
[root@master ~]# route add -net 172.17.2.0 netmask 255.255.255.0 gw 10.211.55.101

# 查看路由
[root@master ~]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.211.55.1     0.0.0.0         UG    100    0        0 eth0
10.211.55.0     0.0.0.0         255.255.255.0   U     100    0        0 eth0
172.17.1.0      0.0.0.0         255.255.255.0   U     0      0        0 docker0
# ********************** 这是刚添加的路由规则 *****************************
172.17.2.0      10.211.55.101   255.255.255.0   UG    0      0        0 eth0
192.168.122.0   0.0.0.0         255.255.255.0   U     0      0        0 virbr0
```
### (5). 在node-1节点添加路由
```
[root@master ~]# route add -net 172.17.1.0 netmask 255.255.255.0 gw 10.211.55.100

# 查看路由
[root@node-1 ~]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.211.55.1     0.0.0.0         UG    100    0        0 eth0
10.211.55.0     0.0.0.0         255.255.255.0   U     100    0        0 eth0
# ************************* 这是刚添加的规则 **************************************
172.17.1.0      10.211.55.100   255.255.255.0   UG    0      0        0 eth0
172.17.2.0      0.0.0.0         255.255.255.0   U     0      0        0 docker0
192.168.122.0   0.0.0.0         255.255.255.0   U     0      0        0 virbr0
```
### (6). 在Master创建容器
```
# 创建容器(linux_1)
[root@master ~]# docker run -it -d --name linux_1 busybox:latest
945181dc3f7b0972ad10a29883456b4664f9363c8ca3a97bfbe209f55340172b

# 创建容器(linux_2)
[root@master ~]# docker run -it -d --name linux_2 busybox:latest
8d3d2791b618fe7298d8d5004e59f8525ddb80a1c87e47701fe68736d72a8f86

# 查看运行中的容器
[root@master ~]# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
8d3d2791b618        busybox:latest      "sh"                3 seconds ago       Up 3 seconds                            linux_2
945181dc3f7b        busybox:latest      "sh"                8 seconds ago       Up 8 seconds                            linux_1
```
### (7). 在node-1节创建容器
```
# 创建容器(linux_3)
[root@node-1 ~]# docker run -it -d --name linux_3 busybox:latest
a0fa30e016f10bb4a30b502e82041252c50925f96c4483bf4eeae73f664e3fd6

# 创建容器(linux_4)
[root@node-1 ~]# docker run -it -d --name linux_4 busybox:latest
5ec5c6eb23ea630f816c7dd53946303f9b836aef9ad56545b3e27440e086d237

# 查看运行中的容器
[root@node-1 ~]# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
5ec5c6eb23ea        busybox:latest      "sh"                3 seconds ago       Up 2 seconds                            linux_4
a0fa30e016f1        busybox:latest      "sh"                7 seconds ago       Up 7 seconds                            linux_3
```
### (8). 在master节点进入容器
```
# 进入容器内部(linux_1)
[root@master ~]# docker exec -it linux_1 sh

# 查看容器内部ip地址
/ # ip addr
6: eth0@if7: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue
    link/ether 02:42:ac:11:01:01 brd ff:ff:ff:ff:ff:ff
    inet 172.17.1.1/24 brd 172.17.1.255 scope global eth0
       valid_lft forever preferred_lft forever

# ping另一个容器(linux_3)的ip
/ # ping 172.17.2.1
PING 172.17.2.1 (172.17.2.1): 56 data bytes
64 bytes from 172.17.2.1: seq=2 ttl=62 time=0.693 ms
```
### (9). 在node-1节点进入容器内部
```
# 在node-1节点进入容器内部
[root@node-1 ~]# docker exec -it linux_3 sh

# 查看容器内部的IP地址
/ # ip addr
6: eth0@if7: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue
    link/ether 02:42:ac:11:02:01 brd ff:ff:ff:ff:ff:ff
    inet 172.17.2.1/24 brd 172.17.2.255 scope global eth0
       valid_lft forever preferred_lft forever

# 在容器内部,ping另一个容器(linux_1)的ip
/ # ping 172.17.1.1
PING 172.17.1.1 (172.17.1.1): 56 data bytes
64 bytes from 172.17.1.1: seq=0 ttl=62 time=0.811 ms
64 bytes from 172.17.1.1: seq=1 ttl=62 time=0.738 ms
```
### (10). 总结
> 通过配置动态路由,可以让两台宿主机里的容器实现互相通信.  