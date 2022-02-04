---
layout: post
title: 'Rancher安装' 
date: 2022-02-03
author: 李新
tags:  Rancher
---

### (1). 概述

### (2). 机器准备

|  机器名称           |           IP  |
|  ----              |         ----  |
| rancher-server     | 10.211.55.100 | 
|  node-1            | 10.211.55.101 | 

### (3). Docker安装
```
[root@rancher-server ~]# yum -y install yum-utils
[root@rancher-server ~]# yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
[root@rancher-server ~]# yum install -y  docker-ce docker-ce-cli containerd.io
[root@rancher-server ~]# systemctl start docker
[root@rancher-server ~]# systemctl enable docker

# 添加用户app到docker组
[root@rancher-server ~]# gpasswd -a app docker
[root@rancher-server ~]# systemctl restart docker

# 创建rancher目录
[root@rancher-server ~]# mkdir -p /opt/rancher_home/rancher
[root@rancher-server ~]# mkdir -p /opt/rancher_home/log/auditlog
[root@rancher-server ~]# chown -R app:app /opt

# 防火墙开启端口
[root@rancher-server ~]# firewall-cmd  --zone=public --add-port=80/tcp --permanent
[root@rancher-server ~]# firewall-cmd  --zone=public --add-port=443/tcp --permanent
[root@rancher-server ~]# firewall-cmd  --reload

```
### (4). Rancher镜像拉取
```
[app@rancher-1 ~]$ docker pull rancher/rancher:v2.4.17  
Using default tag: latest
latest: Pulling from rancher/rancher
a3009803982d: Pull complete
cf9e817c5d35: Pull complete
b380663f8ccc: Pull complete
1ca0e2238656: Pull complete
65086cb458c9: Pull complete
c6d25608690f: Pull complete
6c8ad6da7ce2: Pull complete
6a6940e66f68: Pull complete
b115b1ef2b5b: Pull complete
b4b03dbaa949: Pull complete
aef7deb59b77: Pull complete
0bbf7579a568: Pull complete
eaa5d6336f95: Pull complete
608f536609b9: Pull complete
fcaf65f7937c: Pull complete
e4ea550002d9: Pull complete
9d698b9289d2: Pull complete
caa4144aedf1: Pull complete
Digest: sha256:f411ee37efa38d7891c11ecdd5c60ca73eb03dcd32296678af808f6b4ecccfff
Status: Downloaded newer image for rancher/rancher:latest
docker.io/rancher/rancher:latest
```
### (5). Rancher运行
```
[app@rancher-1 ~]$ docker run -d  --privileged --restart=unless-stopped -p 80:80 -p 443:443 \
-e CATTLE_SYSTEM_CATALOG=bundled \
-e AUDIT_LEVEL=3 \
-v /opt/rancher_home/rancher:/var/lib/rancher \
-v /opt/rancher_home/log/auditlog:/var/log/auditlog \
--name rancher rancher/rancher:v2.4.17 
```
### (6). 查看运行的Rancher容器
[app@rancher-server ~]$ docker ps
CONTAINER ID   IMAGE                     COMMAND           CREATED         STATUS         PORTS                                                                      NAMES
e1a91df8d77b   rancher/rancher:v2.4.17   "entrypoint.sh"   3 minutes ago   Up 3 minutes   0.0.0.0:80->80/tcp, :::80->80/tcp, 0.0.0.0:443->443/tcp, :::443->443/tcp   rancher
### (7). Rancher登录界面
!["Rancher登录页面"](/assets/rancher/imgs/rancher-home.png)   
!["Rancher配置URL"](/assets/rancher/imgs/rancher-set-server-url.png)
!["Rancher首页"](/assets/rancher/imgs/rancher-home2.png)
### (8). 总结
Rancher的版本