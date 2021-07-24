---
layout: post
title: 'Docker Swarm安装与部署(一)' 
date: 2021-07-17
author: 李新
tags:  Docker 
---

### (1). Docker Swarm是什么
Docker Swarm是Docker的集群管理工具.说大白话一点,Docker Swarm类似于K8S,用于管理所有的Docker容器(弹性伸缩).

### (2). Docker Swarm架构
> Docker Swarm集群由管理节点(manager)和工作节点(work node)构成.   
> swarm mananger: 负责整个集群的管理工作包括集群配置、服务管理等所有跟集群有关的工作.    
> work node: 即图中的 available node,主要负责运行相应的服务来执行任务(task).    

!["Docker Swarm架构"](/assets/docker/imgs/docker-swarm-arch.png)

!["Docker Service"](/assets/docker/imgs/docker-swarm-service.png)

### (3). 机器准备

|  机器名称   | IP              | 角色     |
|  ----      | ----            | ----    |
| erp-100    | 10.211.55.100   | manager |
| erp-101    | 10.211.55.101   | worker  |
| erp-102    | 10.211.55.102   | worker  |

### (4). 准备工作
```
# 以下步骤,三台机器都要执行.
# 1. 禁用防火墙
# 如果开启防火墙,则需要在所有节点的防火墙上依次放行2377/tcp（管理端口）、7946/udp（节点间通信端口）、4789/udp（overlay 网络端口）端口.  
>  systemctl disable firewalld.service
>  systemctl stop firewalld.service

# 2. 配置hosts
>  cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
10.211.55.100 erp-100 erp-100
10.211.55.101 erp-101 erp-101
10.211.55.102 erp-102 erp-102

# 3. 安装docker
> yum -y install docker
> systemctl enable docker.service
>  systemctl restart docker


# 4. 查看下宿主机的网络信息
> ifconfig
docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 0.0.0.0

eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.211.55.100  netmask 255.255.255.0  broadcast 10.211.55.255

# ***********************************************************************
# 5. 查看下宿主机的路由信息
# ***********************************************************************
>  route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.211.55.1     0.0.0.0         UG    100    0        0 eth0
10.211.55.0     0.0.0.0         255.255.255.0   U     100    0        0 eth0
192.168.122.0   0.0.0.0         255.255.255.0   U     0      0        0 virbr0
```
### (5). manager节点创建Docker Swarm集群
```
# ***********************************************************************
# 1. 初始化docker swarm
# ***********************************************************************
[root@erp-100 ~]# docker swarm init --advertise-addr 10.211.55.100
# Docker Swarm已经初始化,当前的节点为:manager
Swarm initialized: current node (ykctjoqvzm431tszes4ds50cz) is now a manager.

# 添加:worker节点到当前swarm,请在work节点上运行如下命令.
To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-0in6fxnuvx1ksv6plc023b4elanx2f1wy3eiuzc43rmhtil0ke-1l5ta203lkpawqxyj0d1jprya \
    10.211.55.100:2377

# 添加manager节点到当前swarm,请在work节点上运行如下命令.
To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.


# ***********************************************************************
# 2. 查看node信息
# ***********************************************************************
[root@erp-100 ~]# docker node ls
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
ykctjoqvzm431tszes4ds50cz *  erp-100   Ready   Active        Leader
```
### (6). worker节点,加入Docker Swarm集群
```
# 101节点(work),加入swarm集群
[root@erp-101 ~]# docker swarm join \
>     --token SWMTKN-1-0in6fxnuvx1ksv6plc023b4elanx2f1wy3eiuzc43rmhtil0ke-1l5ta203lkpawqxyj0d1jprya \
>     10.211.55.100:2377
This node joined a swarm as a worker.


# 102节点(work),加入swarm集群
[root@erp-102 ~]# docker swarm join \
>     --token SWMTKN-1-0in6fxnuvx1ksv6plc023b4elanx2f1wy3eiuzc43rmhtil0ke-1l5ta203lkpawqxyj0d1jprya \
>     10.211.55.100:2377
This node joined a swarm as a worker.
```
### (7). 查看swarm集群状态
```
[root@erp-100 ~]# docker node ls
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
lv3aruz9u70v7rbs5zgotkeyg    erp-102   Ready   Active
ooydlst3kukvy0ysq75johokh    erp-101   Ready   Active
ykctjoqvzm431tszes4ds50cz *  erp-100   Ready   Active        Leader
```
### (8). 创建service
```
# 1. 创建service时,需要nginx镜像,防止镜像下载过慢,提前拉取镜.
[root@erp-102 ~]# docker pull nginx
[root@erp-102 ~]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
docker.io/nginx     latest              08b152afcfae        44 hours ago        133 MB

# 2. 查看docker网络信息
[root@erp-100 ~]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
5f4e36927e4a        bridge              bridge              local
43ecb3f9c1b2        docker_gwbridge     bridge              local
ea0745ad920e        host                host                local
wd84uip49n0d        ingress             overlay             swarm
2bedf4bd7822        none                null                local

# 3. 查看网络ingress信息
[root@erp-100 ~]# docker network inspect ingress
"Containers": {
	"74f343fdc8fb06718c96287dd336fc356778d1e17cb7744434c82440cfd6367d": {
		"Name": "mynginx.3.edq8kzv5w2enaxsl021effaph",
		"EndpointID": "cd393d4d14d7ad513e4fdff958ac450c3c4035be529d39ec67c0357065009aef",
		"MacAddress": "02:42:0a:ff:00:09",
		"IPv4Address": "10.255.0.9/16",
		"IPv6Address": ""
	},
	"ingress-sbox": {
		"Name": "ingress-endpoint",
		"EndpointID": "cd007a4a2d5ace0cdfca857d66d5b3b6a517e2156e3f167718a80a2f2bb9c7bd",
		"MacAddress": "02:42:0a:ff:00:03",
		"IPv4Address": "10.255.0.3/16",
		"IPv6Address": ""
	}
}

# 4. 创建overlay网络
#    在创建service时,如果你不指定,则,网络是:ingress
[root@erp-100 ~]# docker network create --driver overlay --opt encrypted --subnet 10.10.10.0/24 nginx_network
w09ts43g3607rdl68thy3aw00

# 5.查看网络状态
[root@erp-100 ~]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
5f4e36927e4a        bridge              bridge              local
43ecb3f9c1b2        docker_gwbridge     bridge              local
ea0745ad920e        host                host                local
wd84uip49n0d        ingress             overlay             swarm
w09ts43g3607        nginx_network       overlay             swarm
2bedf4bd7822        none                null                local

# 6. 在master节点上执行如下命令来创建名为mynginx的service,让其有2份nginx容器副本分配到集群中去,并指定自己创建的网络(nginx_network)
[root@erp-100 ~]# docker service create --replicas 2 --network nginx_network  -p 8080:80 --name mynginx  nginx
tu9oyus6c5ma58mk9zih0agwf

# 7. 查看所有的service.
[root@erp-100 ~]# docker service ls
ID            NAME     MODE        REPLICAS  IMAGE
wkko7kvn6fll  mynginx  replicated  2/2       nginx:latest


# 8. 查看service(mynginx),运行状态.
[root@erp-100 ~]# docker service ps mynginx
ID            NAME       IMAGE         NODE     DESIRED STATE  CURRENT STATE           ERROR  PORTS
hqn9x54cthro  mynginx.1  nginx:latest  erp-101  Running        Running 41 seconds ago
7cytp3g6sfdx  mynginx.2  nginx:latest  erp-100  Running        Running 41 seconds ago


# 9. 删除servie
# [root@erp-100 ~]# docker service rm mynginx
# mynginx
```
### (9). 测试访问service
```
# 因为,只创建了两个容器,我有三台宿主机,访问三台宿主机是否能正确
# 1. 访问:10.211.55.100
lixin-macbook:~ lixin$ curl -vvv http://10.211.55.100:8080
* Closing connection -1
curl: (3) URL using bad/illegal format or missing URL
*   Trying 10.211.55.100...
* TCP_NODELAY set
* Connected to 10.211.55.100 (10.211.55.100) port 8080 (#0)
> GET / HTTP/1.1
> Host: 10.211.55.100:8080
> User-Agent: curl/7.64.1
> Accept: */*


# 2. 访问:10.211.55.101
lixin-macbook:~ lixin$ curl -vvv http:// 10.211.55.101:8080
* Closing connection -1
curl: (3) URL using bad/illegal format or missing URL
*   Trying 10.211.55.101...
* TCP_NODELAY set
* Connected to 10.211.55.101 (10.211.55.101) port 8080 (#0)
> GET / HTTP/1.1
> Host: 10.211.55.101:8080
> User-Agent: curl/7.64.1
> Accept: */*


# 3. 访问:10.211.55.102
lixin-macbook:~ lixin$ curl -vvv http:// 10.211.55.102:8080
* Closing connection -1
curl: (3) URL using bad/illegal format or missing URL
*   Trying 10.211.55.102...
* TCP_NODELAY set
* Connected to 10.211.55.102 (10.211.55.102) port 8080 (#0)
> GET / HTTP/1.1
> Host: 10.211.55.102:8080
> User-Agent: curl/7.64.1
> Accept: */*

# ***********************************************************
# 从上面的结论来看,只要创建service,就会在所有的宿主机上创建相应的port,并监听
# ***********************************************************
```
### (10). 动态伸缩service
```
# 1. 查看service(mynginx)状态
[root@erp-100 ~]# docker service ps mynginx
ID            NAME       IMAGE         NODE     DESIRED STATE  CURRENT STATE          ERROR  PORTS
hqn9x54cthro  mynginx.1  nginx:latest  erp-101  Running        Running 8 minutes ago
7cytp3g6sfdx  mynginx.2  nginx:latest  erp-100  Running        Running 8 minutes ago

# 2. 动态伸缩(scale)
[root@erp-100 ~]# docker service  scale mynginx=4
mynginx scaled to 4

# 3. 再次查看service(mynginx),变成了4个
[root@erp-100 ~]# docker service ps mynginx
ID            NAME       IMAGE         NODE     DESIRED STATE  CURRENT STATE           ERROR  PORTS
hqn9x54cthro  mynginx.1  nginx:latest  erp-101  Running        Running 12 minutes ago
7cytp3g6sfdx  mynginx.2  nginx:latest  erp-100  Running        Running 12 minutes ago
nnqpqe8uy550  mynginx.3  nginx:latest  erp-102  Running        Running 3 minutes ago
s68z4bdz114q  mynginx.4  nginx:latest  erp-102  Running        Running 3 minutes ago

# 4. 动态伸缩(update)
[root@erp-100 ~]# docker service update --replicas 3 mynginx
mynginx

# 5. 再次查看service(mynginx)信息,变成了3个
[root@erp-100 ~]# docker service ps mynginx
ID            NAME       IMAGE         NODE     DESIRED STATE  CURRENT STATE           ERROR  PORTS
hqn9x54cthro  mynginx.1  nginx:latest  erp-101  Running        Running 13 minutes ago
7cytp3g6sfdx  mynginx.2  nginx:latest  erp-100  Running        Running 13 minutes ago
s68z4bdz114q  mynginx.4  nginx:latest  erp-102  Running        Running 4 minutes ago
```
### (11). 查看网络状态
```
# 1. 查看宿主机上(erp-100)的网络信息,发现有一个容器,IP地址为:10.10.10.4
[root@erp-100 ~]# docker network inspect nginx_network
 "Containers": {
	"1a7ec863ff9408640f9b252e2f410b6b936d0b3ad5700343de79e218c54ed502": {
		"Name": "mynginx.2.nfieat3926njm8b7tfocv53tj",
		"EndpointID": "c4bce32fda04b52aff9f01327620b29ea9cbef2760954a3bde3cb0d5a138fb0e",
		"MacAddress": "02:42:0a:0a:0a:04",
		"IPv4Address": "10.10.10.4/24",
		"IPv6Address": ""
	}
}

# 2. 查看宿主机上(erp-101)的网络信息,发现有一个容器,IP地址为:10.10.10.2
[root@erp-101 ~]# docker network inspect nginx_network
"Containers": {
	"58d86999e05386512340be0f45cc3366037833752d367f3f15325bf574ba130b": {
		"Name": "mynginx.1.jsgzk82531bm1mx3uhup6v0bb",
		"EndpointID": "edddb7e9e9a50f957536e08e70cda5ebcc909287f1572762d7eae9de5f8cb9d7",
		"MacAddress": "02:42:0a:0a:0a:03",
		"IPv4Address": "10.10.10.3/24",
		"IPv6Address": ""
	}
}

# 3. 查看宿主机上(erp-102)的网络信息,发现有一个容器,IP地址为:10.10.10.5
[root@erp-102 ~]# docker network inspect nginx_network
"Containers": {
	"d94b9823dcbad04c6b6a22a507dd2b283adb2c32f648cc2964efdba2274bbe75": {
		"Name": "mynginx.3.v6p1q864b6pgj00xfxoof7lfx",
		"EndpointID": "4b78c37406cdac61655abcaa1305d2c59f63bb74c4abb8a3f00438ba9612bd74",
		"MacAddress": "02:42:0a:0a:0a:05",
		"IPv4Address": "10.10.10.5/24",
		"IPv6Address": ""
	}
}
```
### (12). 测试进入容器内部互相ping
```
# 1. 查看当前宿主机上有哪些活动的容器
[root@erp-100 ~]# docker ps
CONTAINER ID        IMAGE                                                                           COMMAND                  CREATED             STATUS              PORTS               NAMES
1a7ec863ff94        nginx@sha256:8f335768880da6baf72b70c701002b45f4932acae8d574dedfddaf967fc3ac90   "/docker-entrypoin..."   9 minutes ago       Up 9 minutes        80/tcp              mynginx.2.nfieat3926njm8b7tfocv53tj


# 2. 进入容器内部,查看IP地址信息(能够看到这个容器的ip[10.10.10.4])
[root@erp-100 ~]# docker exec -it 1a7ec863ff94 /bin/bash
root@1a7ec863ff94:/# cat /etc/hosts
127.0.0.1       localhost
::1     localhost ip6-localhost ip6-loopback
10.10.10.4      1a7ec863ff94

# 3. 在容器内部,测试访问,其它容器内部的nginx
# 10.10.10.2
root@1a7ec863ff94:/# curl -vvv http://10.10.10.2
* Expire in 0 ms for 6 (transfer 0x562e8cf98fb0)
*   Trying 10.10.10.2...
* TCP_NODELAY set
* Expire in 200 ms for 4 (transfer 0x562e8cf98fb0)
* Connected to 10.10.10.2 (10.10.10.2) port 80 (#0)
> GET / HTTP/1.1
> Host: 10.10.10.2

#  10.10.10.5
root@1a7ec863ff94:/# curl -vvv http://10.10.10.5
* Expire in 0 ms for 6 (transfer 0x55e707324fb0)
*   Trying 10.10.10.5...
* TCP_NODELAY set
* Expire in 200 ms for 4 (transfer 0x55e707324fb0)
* Connected to 10.10.10.5 (10.10.10.5) port 80 (#0)
> GET / HTTP/1.1
> Host: 10.10.10.5
> User-Agent: curl/7.64.0
> Accept: */*
```
### (13). 宿主机是否能访问容器呢
```
# 宿主机上,能否访问容器内的ip地址呢?
# 总结:在宿主机上,不能访问到容器内部的IP
# 因为,docker容器内部的IP会通过:docker_gwbridge进行转发,然后,docker_gwbridge又通过::eth0转发.
# docker--> docker_gwbridge  --> eth0
[root@erp-100 ~]# ping 10.10.10.5
PING 10.10.10.5 (10.10.10.5) 56(84) bytes of data.
--- 10.10.10.5 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1001ms

# 1. 安装ip命令
root@1a7ec863ff94:/# apt update && apt install -y iproute2
#    安装ping命令
root@74f343fdc8fb:/# apt-get update && apt-get install iputils-ping && apt-get install dnsutils

# 2. 查看ip地址
root@1a7ec863ff94:/# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
26: eth1@if27: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1424 qdisc noqueue state UP group default
    link/ether 02:42:0a:0a:0a:04 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet 10.10.10.4/24 scope global eth1
       valid_lft forever preferred_lft forever
    inet 10.10.10.2/32 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::42:aff:fe0a:a04/64 scope link
       valid_lft forever preferred_lft forever
28: eth2@if29: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:12:00:03 brd ff:ff:ff:ff:ff:ff link-netnsid 2
    inet 172.18.0.3/16 scope global eth2
       valid_lft forever preferred_lft forever
    inet6 fe80::42:acff:fe12:3/64 scope link
       valid_lft forever preferred_lft forever
30: eth0@if31: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
    link/ether 02:42:0a:ff:00:08 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.255.0.8/16 scope global eth0
       valid_lft forever preferred_lft forever
    inet 10.255.0.6/32 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:aff:feff:8/64 scope link
       valid_lft forever preferred_lft forever

# 3.查看路由信息(172.18.0.0网段的出口为:0.0.0.0,而0.0.0.0的出口是:10.211.55.1)
[root@erp-100 ~]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.211.55.1     0.0.0.0         UG    100    0        0 eth0
10.211.55.0     0.0.0.0         255.255.255.0   U     100    0        0 eth0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
172.18.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker_gwbridge
192.168.122.0   0.0.0.0         255.255.255.0   U     0      0        0 virbr0
```
### (14). 总结
> 总体来说Docker Swarm相比K8S还是比较简单的,特别是,机器数量不多的情况下,用Swarm会比K8S更节省.  
