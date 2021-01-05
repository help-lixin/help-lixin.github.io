---
layout: post
title: 'Docker 四种网络模型'
date: 2020-11-01
author: 李新
tags: Docker
---


### (1) 注意,以下内容皆参考如下链接

["Docker四种网络模型"](https://zhuanlan.zhihu.com/p/98788162)

### (2). NAT(bridge)
> 这是Docker默认的网络模式,在启动Docker时无须参数指定.   
> 当Docker进程启动时,会在宿主机上创建一个名为docker0的虚拟网桥,此宿主机上启动的Docker容器会连接到这个虚拟网桥上.    
> 虚拟网桥的工作方式和物理交换机类似,这样宿主机上的所有容器就通过交换机连在了一个二层网络中.    

!["bridge模式"](https://pic3.zhimg.com/80/v2-1d0f6cf1b79010467fe9dcd55465384e_1440w.jpg)


```
lixin-macbook:~ lixin$ docker run -it --rm  centos:centos7.1 /bin/bash 
[root@4bca21e4ad53 /]# ifconfig 
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.2  netmask 255.255.0.0  broadcast 172.17.255.255
        ether 02:42:ac:11:00:02  txqueuelen 0  (Ethernet)
```
### (3). None
> Docker容器拥有自己的Network Namespace,但是,并不为Docker容器进行任何网络配置.    
> 也就是说,这个Docker容器没有网卡、IP、路由等信息.   
> 比如:一些计算资源不需要网络通信.

!["None"](https://pic2.zhimg.com/80/v2-9307da908b8f0c641cf721d2abb774b5_1440w.jpg)

```
lixin-macbook:~ lixin$ docker run -it --rm --net=none lixinhelp/centos:centos7 /bin/bash 
[root@45a4ae218d9c /]# ifconfig 
lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```
### (4). Host
> 如果启动容器的时候使用host模式,那么这个容器将不会获得一个独立的Network Namespace.   
> 而是和宿主机共用一个Network Namespace,容器将不会虚拟出自己的网卡,配置自己的IP等.而是使用宿主机的IP和端口.但是,容器的其他方面,如文件系统、进程列表等还是和宿主机隔离的.  

!["Host"](https://pic1.zhimg.com/80/v2-8195e42c05a0be0dc1cf5803b63ebcd8_1440w.jpg)

```
lixin-macbook:~ lixin$ docker run -it --rm --net=host lixinhelp/centos:centos7 /bin/bash 
[root@docker-desktop /]# ifconfig 
docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
        inet6 fe80::42:8aff:fee5:d0d7  prefixlen 64  scopeid 0x20<link>
        ether 02:42:8a:e5:d0:d7  txqueuelen 0  (Ethernet)
        RX packets 6290  bytes 254993 (249.0 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 9489  bytes 12459416 (11.8 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.65.3  netmask 255.255.255.0  broadcast 192.168.65.255
        inet6 fe80::50:ff:fe00:1  prefixlen 64  scopeid 0x20<link>
        ether 02:50:00:00:00:01  txqueuelen 1000  (Ethernet)
        RX packets 99860  bytes 137638521 (131.2 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 56096  bytes 19944803 (19.0 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 2  bytes 140 (140.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 2  bytes 140 (140.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```
### (5). Container
> 这个模式指定新创建的容器和已经存在的一个容器共享一个 Network Namespace,而不是和宿主机共享.   
> 新创建的容器不会创建自己的网卡,配置自己的 IP,而是和一个指定的容器共享 IP、端口范围等.同样,两个容器除了网络方面,其他的如文件系统、进程列表等还是隔离的.   
> 两个容器的进程可以通过 lo 网卡设备通信.   

!["Container"](https://pic4.zhimg.com/80/v2-6a7315a99af7b45e5dae554f0a5d6e33_1440w.jpg)

```
# 创建nginx容器
lixin-macbook:~ lixin$ docker run -d --rm  --name nginx nginx:laster  
2868b22b586042924024d5f7098a24e5944324c02f9bc0519a8ad1f0c4fc8fcc

# 查看容器是否运行
lixin-macbook:~ lixin$ docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS         PORTS     NAMES
2868b22b5860   nginx:laster   "/bin/sh -c /entrypo…"   3 seconds ago   Up 2 seconds             nginx

# 创建centos容器,指定网络模式(两个容器公用同一个网络设备,即IP信息)
lixin-macbook:~ lixin$ docker run -it --rm --name nginx-test --net=container:2868b22b5860 lixinhelp/centos:centos7 /bin/bash 

# 查看开启了哪些端口
[root@2868b22b5860 /]# netstat -tlnp
	Active Internet connections (only servers)
	Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
	tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      -                   
	tcp6       0      0 :::80                   :::*                    LISTEN      -                   

# 测试访问
[root@2868b22b5860 /]# curl  http://localhost:80
	<html>
		<head>
			<title>403 Forbidden</title>
		</head>
		<body>
			<center><h1>403 Forbidden</h1></center>
			<hr><center>nginx/1.16.1</center>
		</body>
	</html>
```