---
layout: post
title: 'Zerotier自建' 
date: 2023-04-15
author: 李新
tags:  Zerotier VPN WireGuard
---

### (1). 概述

最近因为自己通过ESXI搭服务器,同事分散在各处办公,又要解决通信问题,对SD WAN稍有研究,此处,特意记录:Zerotier安装过程

### (2). Zerotier根服务器与UI界面安装
```
#!/bin/bash 
# centos自动安装zerotier 并设置的为planet服务器
# addr服务器公网ip+port
# ip=`wget http://ipecho.net/plain -O - -q ; echo`
 ip=192.168.1.14 
 addr=$ip/9993

 echo "********************************************************************************************************************"
 echo "**********centos自动安装zerotier 并设置的为planet服务器 火木木制作 放在root目录执行**********************************"
curl -s https://install.zerotier.com | bash
 yum install gcc gcc-c++ -y
 yum install git -y
cd /opt && git clone -v https://gitee.com/opopop880/ZeroTierOne.git --depth 1 
cd /var/lib/zerotier-one && zerotier-idtool initmoon identity.public > moon.json
cd / && git clone https://gitee.com/opopop880/docker-zerotier-planet.git
mv ./docker-zerotier-planet ./app
  echo "{
  \"stableEndpoints\": [
    \"$ip/9993\"
  ]
}
" >/app/patch/patch.json
cd /app/patch && python3 patch.py 
cd /var/lib/zerotier-one && zerotier-idtool genmoon moon.json && mkdir moons.d && cp ./*.moon ./moons.d
cd /opt/ZeroTierOne/attic/world/ && sh build.sh
sleep 8s
cd /opt/ZeroTierOne/attic/world/ && ./mkworld
mkdir /app/bin -p && cp world.bin /app/bin/planet
cd /app/bin/
\cp -r ./planet /var/lib/zerotier-one/
\cp -r ./planet /root
systemctl restart zerotier-one.service
wget https://gitee.com/opopop880/ztncui/attach_files/932633/download/ztncui-0.8.6-1.x86_64.rpm
rpm -ivh ztncui-0.8.6-1.x86_64.rpm
cd /opt/key-networks/ztncui/
echo 'HTTP_PORT=3443' >.env
echo 'NODE_ENV=production' >>.env
echo 'HTTP_ALL_INTERFACES=true' >>.env
systemctl restart ztncui
rm -rf /opt/ZeroTierOne
echo "**********安装成功*********************************************************************************"
```
### (3). Zerotier创建网络

> 登录zerotier(admin/password)


!["Zerotier创建网络"](/assets/zerotier/imgs/zerotier-ui-network.png)

### (4). linux client加入Zerotier网络
```
[root@jenkins ~]# curl -s https://install.zerotier.com | sudo bash

[root@jenkins ~]# zerotier-cli info
200 info 8d211494d1 1.10.5 ONLINE

[root@jenkins ~]# systemctl enable zerotier-one.service


# 从自建根服务器拷贝planet文件到客户端进行替换
[root@jenkins ~]# scp root@192.168.1.14:/root/planet /var/lib/
[root@jenkins ~]# systemctl restart zerotier-one

[root@jenkins ~]# systemctl status zerotier-one
● zerotier-one.service - ZeroTier One
   Loaded: loaded (/usr/lib/systemd/system/zerotier-one.service; enabled; vendor preset: disabled)
   Active: active (running) since Sat 2023-04-15 21:58:41 CST; 20s ago
 Main PID: 25192 (zerotier-one)
   CGroup: /system.slice/zerotier-one.service
           └─25192 /usr/sbin/zerotier-one

Apr 15 21:58:41 jenkins systemd[1]: Started ZeroTier One.

# 加入网络
[root@jenkins ~]# zerotier-cli join ec7246f3e1eadab6
200 join OK

# 查看网络信息(并未分配ip)
[root@jenkins ~]# ip addr
3: ztd6j2ykkx: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 2800 qdisc pfifo_fast state UNKNOWN group default qlen 1000
    link/ether b6:57:cb:f5:67:97 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::b457:cbff:fef5:6797/64 scope link 
       valid_lft forever preferred_lft forever
```
### (4). Zerotier授权通过

!["Zerotier授权通过"](/assets/zerotier/imgs/zerotier-member.png)

### (5). Linux client查看网络状态
```
[root@jenkins ~]# ip addr
3: ztd6j2ykkx: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 2800 qdisc pfifo_fast state UNKNOWN group default qlen 1000
    link/ether b6:57:cb:f5:67:97 brd ff:ff:ff:ff:ff:ff
    inet 10.249.2.207/24 brd 10.249.2.255 scope global ztd6j2ykkx
       valid_lft forever preferred_lft forever
    inet6 fe80::b457:cbff:fef5:6797/64 scope link 
       valid_lft forever preferred_lft forever
```
### (6). 总结
Zerotier是不建议自己搭建根服务器的,不过Github上总有大神,可以根据源码进行更改,并配置出UI界面. 