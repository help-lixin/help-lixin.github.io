---
layout: post
title: 'Linux SSH隧道(正向代理)'
date: 2017-06-04
author: 李新
tags: Linux
---

### (1). 前言

### (2). 机器准备

|  环境    | IP             | Port   |
|  ----   | ----           | ----   |
| 公网     | 47.119.169.76  | 3306   |
| 内网     | 172.17.12.223  | 3800   |

### (3). 内网机器配置免密钥
```
# 1. 内网机器(172.17.12.223)生成密钥
lixin-macbook:~ lixin$ ssh-keygen

# 2. 会在(172.17.12.223)用户的目录下生成密钥对
lixin-macbook:~ lixin$ ll ~/.ssh
-rw-------   1 lixin  staff  1675  4 15  2020 id_rsa
-rw-------   1 lixin  staff   401  4 15  2020 id_rsa.pub

# 3. 拷贝内网机器(172.17.12.223)生成的公钥到远程机器上
lixin-macbook:~ lixin$ ssh-copy-id root@47.119.169.76
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/Users/lixin/.ssh/id_rsa.pub"
The authenticity of host '47.119.169.76 (47.119.169.76)' can't be established.
ECDSA key fingerprint is SHA256:gwtG1422w2+CAyM5UngIAmcp1sy8SSM8GPXLaWUTvrQ.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
root@47.119.169.76's password:
Number of key(s) added:        1
Now try logging into the machine, with:   "ssh 'root@47.119.169.76'"
and check to make sure that only the key(s) you wanted were added.

# 4. 测试ssh连接
lixin-macbook:~ lixin$ ssh root@47.119.169.76
Last login: Wed Jun  2 19:09:28 2021 from 219.133.101.169
Welcome to Alibaba Cloud Elastic Compute Service !
```
### (4). 外网机器允许(3306)端口通行(略)

### (5). 外网机器配置(sshd_config)
> 注意:这一步一定要做,否则,你会发现:监听的地址是:127.0.0.1,而不是:0.0.0.0,倒至你无法访问.  

```
# 1. 修改外网机器(47.119.169.76)的SSHD配置
[root@lixin ~]# vi /etc/ssh/sshd_config
# 修改这个内容为yes
GatewayPorts yes

# 2. 重启sshd(47.119.169.76)
[root@lixin ~]# systemctl restart sshd.service
```
### (6). 内网通过SSH隧道与外网建立连接
```
# ssh参数介绍
# -f 后台执行ssh指令
# -N 不执行远程指令
# -C 允许压缩数据
# -R 将远程主机(服务器)的某个端口转发到本地端指定机器的指定端口
# -L 将本地机(客户机)的某个端口转发到远端指定机器的指定端口
# -p 指定远程主机的SSH端口

# 正向代理:
# 在本机,启动一个3800端口,映射到:47.119.169.76的3306端口.
lixin-macbook:~ lixin$ ssh -fNCL  0.0.0.0:3800:47.119.169.76:3306 root@47.119.169.76
```
### (7). 内网机器查看是否监听端口成功
```
# 1.查看(172.17.12.223)端口是否监听成功:
lixin-macbook:~ lixin$ lsof -i tcp:3800
COMMAND  PID  USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
ssh     5676 lixin    4u  IPv4 0x3574c3ddb0431051      0t0  TCP *:pwgpsi (LISTEN)
```
### (8). 使用autossh
> 上面通过ssh配置的正向代理的方式是不稳定的,这种ssh链接会因为超时而关闭,如果关闭了,那么内网连通外网的通道就无法维持了,为此我们需要另外的方法来提供稳定的ssh反向代理隧道工具(autossh).

```
# 1. Mac安装ssh
lixin-macbook:~ lixin$ brew install autossh

# 2. 通过autossh启动
# 参数说明:
#    -M   : 使用内网主机的55555端口监视SSH连接状态,连接出问题了会自动重连.
#    -N   : 不执行远程命令
#    -L   : 将内网主机的某个端口的请求转发公网的某个端口上.
lixin-macbook:~ lixin$ autossh -M 55555 -NfL 3800:47.119.169.76:3306 root@47.119.169.76

# 3. 测试是否连接成功
lixin-macbook:~ lixin$ mysql -h 127.0.0.1 -P 3800 -u lixin -p
Enter password:

Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 6
# *********************************************
//  注意:这里是MariaDB
Server version: 5.5.68-MariaDB MariaDB Server
# *********************************************
Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.
Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
mysql>
```
### (9). 总结
> 通过SSH正向隊道代理,可以实现内网和外网的穿透(通过访问内网的IP:PORT,透传到外网的IP:PORT).  