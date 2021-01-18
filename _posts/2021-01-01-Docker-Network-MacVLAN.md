---
layout: post
title: 'Docker 跨主机路由之MacVLAN'
date: 2021-01-01
author: 李新
tags: Docker
---

### (1). 环境准备

|  宿主IP         | 容器网段       | 主机名称     |
|  ----          | ----          |   ----      |
| 10.211.55.100  | 172.17.0.1/24 |  master     |
| 10.211.55.101  | 172.17.0.1/24 |  node-1     |

### (2). 开启网卡的混杂模式
> 两台宿主机器上的主网卡都要开启网卡混杂模式:    
> ip link set eth0 promisc on(开启)     
> ip link set eth0 promisc off(关闭)    

```
# 开启网卡混杂模式(PROMISC)
$ ip link set eth0 promisc on
$ ifconfig eth0
eth0: flags=4419<UP,BROADCAST,RUNNING,PROMISC,MULTICAST>  mtu 1500
```
### (3). 在两台宿主机上,创建一个MacVLAN网络
> 注意:掩码/网关 都要和宿主机上的主网卡要一样的.   

```
# route -n 可查询本机的网关(Destionation=0.0.0.0)
# 创建MacVLAN(macvlantest)网络
$ docker network create --driver macvlan --subnet 10.211.55.0/24 --gateway 10.211.55.1 -o parent=eth0 macvlantest

# 查看是否添加成功
$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
1238f6eff72b        macvlantest         macvlan             local
```
### (4). 在master节点创建容器,并测试
```
# 注意:
# 指定: --net macvlantest
# 指定: --ip 10.211.55.10
[root@master ~]# docker run -d -it --rm --name linux_1 --net macvlantest --ip 10.211.55.10 busybox:latest
5968b159f08209260e75903f168939f2f5260453e368d53499ca28f2aefe8aea

# 查看容器
[root@master ~]# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
5968b159f082        busybox:latest      "sh"                3 seconds ago       Up 3 seconds                            linux_1

# 进入容器内部进行测试
[root@master ~]# docker exec -it linux_1 sh
# 查看容器当前的IP地址
/ # ip addr
6: eth0@if2: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue
    link/ether 02:42:0a:d3:37:0a brd ff:ff:ff:ff:ff:ff
    inet 10.211.55.10/24 brd 10.211.55.255 scope global eth0
       valid_lft forever preferred_lft forever


# 在容器内部ping另一台宿主机(node-1)的IP
/ # ping 10.211.55.101
PING 10.211.55.101 (10.211.55.101): 56 data bytes
64 bytes from 10.211.55.101: seq=0 ttl=64 time=0.750 ms
64 bytes from 10.211.55.101: seq=1 ttl=64 time=0.567 ms
```
### (5). 在node-1节点创建容器,并测试
```
# 创建容器
# 指定: --net macvlantest
# 指定: --ip 10.211.55.11
[root@node-1 ~]# docker run -d -it --rm --name linux_2 --net macvlantest --ip 10.211.55.11 busybox:latest
fb56cfe3e1efc056c37a0ce1b7dd278114c9a84318a95330361cf47d5f6fd9c5

# 查看容器
[root@node-1 ~]# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
fb56cfe3e1ef        busybox:latest      "sh"                4 seconds ago       Up 2 seconds                            linux_2

# 进入容器内部
[root@node-1 ~]# docker exec -it linux_2 sh

# 查看容器的IP
/ # ip addr
6: eth0@if2: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue
    link/ether 02:42:0a:d3:37:0b brd ff:ff:ff:ff:ff:ff
    inet 10.211.55.11/24 brd 10.211.55.255 scope global eth0
       valid_lft forever preferred_lft forever

# 在容器内部ping另一个容器的IP
/ # ping 10.211.55.10
PING 10.211.55.10 (10.211.55.10): 56 data bytes
64 bytes from 10.211.55.10: seq=1 ttl=64 time=0.305 ms
```
### (6). 总结
> MacVLAN的感觉就是针对一个物理网卡,设置多个mac地址,针对每一个mac地址都可以单独配置IP. 
