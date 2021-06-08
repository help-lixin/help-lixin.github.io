---
layout: post
title: 'Centos 7 运用iptables NAT上网'
date: 2017-10-04
author: 李新
tags: Linux
---

### (1). 前言
> 在生产上,一般只有堡垒机和Nginx服务器有外网IP,其余机器,有时需要上网(比如yum安装软件或升级),没有外网IP很不方便,可是,每台机器都有外网IP又很浪费资源.    
> 能否:让内网所有的机器,在进行访问外网时,都转发给有外网IP流量的机器,让它(堡垒机)帮忙实现网络代理呢?  

### (2). 机器准备

|  机器名称   | 内网IP         |     内网IP     |     描述      |
|  ----      | ----          |   ----        |      ----      |
|  tomcat-1  | 10.211.55.100 | 10.37.129.7   |    能上外网     |
|  tomcat-2  |               | 10.37.129.6   |    不能上外网   |


### (3). 检查机器信息

```
# 1. 检查tomcat-1 IP地址信息(10.211.55.100/10.37.129.7)
#       tomcat-1有两张网卡.
[root@tomcat-1 ~]# ip addr
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:1c:42:c8:f8:33 brd ff:ff:ff:ff:ff:ff
    inet 10.211.55.100/24 brd 10.211.55.255 scope global noprefixroute eth0
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:1c:42:8e:4d:48 brd ff:ff:ff:ff:ff:ff
    inet 10.37.129.7/24 brd 10.37.129.255 scope global noprefixroute dynamic eth1

# 2.检查tomcat-2 IP地址信息
[root@tomcat-2 ~]# ip addr
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:1c:42:be:bf:93 brd ff:ff:ff:ff:ff:ff
    inet 10.37.129.6/24 brd 10.37.129.255 scope global noprefixroute dynamic eth0
	   
# 3. 要先证实:tomcat-1可以与tomcat-2互相方问.
#    tomcat-1可以访问外网
#    tomcat-2不可以访问外网
[root@tomcat-1 ~]# ping www.baidu.com
PING www.a.shifen.com (183.232.231.174) 56(84) bytes of data.
64 bytes from 183.232.231.174 (183.232.231.174): icmp_seq=1 ttl=128 time=51.4 ms

[root@tomcat-2 ~]# ping www.baidu.com
connect: Network is unreachable
```
### (4). 准备工作
```
# 停用firewalld
> systemctl stop firewalld
# 禁用firewalld
> systemctl mask firewalld
# 安装(升级)iptables
> yum install -y iptables
> yum update iptables
# 添加规则,先允许所有的连接,否则,呆会连接不上去
> iptables -P INPUT ACCEPT
# 保存iptables规则信息
> service iptables save
# 启用iptables开机启动
> systemctl enable iptables.service
# 启动iptables
> systemctl start iptables.service
# 查看iptables状态
> systemctl status iptables.service
```
### (5). tomcat-1配置
```
# 1. 允许IP转发功能
[root@tomcat-1 ~]# echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.conf
[root@tomcat-1 ~]# sysctl -p
net.ipv4.ip_forward = 1

# 2. 添加SNAT规则
#    把源地址为:10.37.129.0/24的网段,转换成:10.211.55.100这个公网IP出去
[root@tomcat-1 ~]# iptables -t nat -A POSTROUTING -s 10.37.129.0/24 -j SNAT --to 10.211.55.100

# 清空规则时,要指定-t
# [root@tomcat-1 ~]# iptables -t nat -F   

# 3. 保存iptables
[root@tomcat-1 ~]# service iptables save
[root@tomcat-1 ~]# service iptables restart
```
### (6). tomcat-2配置
```
# 1. 查看路由表
[root@tomcat-2 ~]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
10.37.129.0     0.0.0.0         255.255.255.0   U     100    0        0 eth0
192.168.122.0   0.0.0.0         255.255.255.0   U     0      0        0 virbr0

# 2. 添加能上网的机器为网关
[root@tomcat-2 ~]#  route add default gw 10.37.129.7

# 3. 再次查看网关信息
[root@tomcat-2 ~]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.37.129.7     0.0.0.0         UG    0      0        0 eth0
10.37.129.0     0.0.0.0         255.255.255.0   U     100    0        0 eth0
192.168.122.0   0.0.0.0         255.255.255.0   U     0      0        0 virbr0
```
### (7). 测试
```
[root@tomcat-2 ~]# ping www.baidu.com
64 bytes from 14.215.177.38 (14.215.177.38): icmp_seq=1 ttl=127 time=8.60 ms
64 bytes from 14.215.177.38 (14.215.177.38): icmp_seq=2 ttl=127 time=11.2 ms
```
