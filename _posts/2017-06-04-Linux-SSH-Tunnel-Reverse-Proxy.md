---
layout: post
title: 'Linux SSH隧道(反向代理)'
date: 2017-06-04
author: 李新
tags: Linux
---

### (1). 前言
> 要对接微信(支付宝)的支付功能,支付后有一个回调功能,开发人员期望:这个回调的请求能转发到自己的机器上(方便联调),而公司的IP地址是动态的,这时候,就要用到:内网穿透功能了.

### (2). 机器准备

|  环境    | IP             | Port   |
|  ----   | ----           | ----   |
| 公网     | 47.119.169.76  | 8081   |
| 内网     | 172.17.12.223  | 8080   |

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
### (4). 外网机器允许(8081)端口通行(略)

### (5). 外网机器配置(sshd_config)
> 注意:这一步一定要做,否则,你会发现:监听的地址是:127.0.0.1,而不是:0.0.0.0,导致你无法远程访问(只能本地访问).  

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

# 反向代理:
# 在公网:47.119.169.76机器上,开启一个8081的端口与内网(本机)的8080端口进行映射
lixin-macbook:~ lixin$ ssh -fNCR  *:8081:localhost:8080 root@47.119.169.76
```
### (7). 外网机器查看是否监听端口成功
```
# 查看(47.119.169.76)端口是否监听成功:
[root@lixin ~]# ss -tlnp|grep 8081
LISTEN     0      128    0.0.0.0:8081                     *:*                   users:(("sshd",pid=1517,fd=8))

# 在外网的机器(47.119.169.76)上测试访问
[root@lixin ~]# curl http://47.119.169.76:8081/hello
Hello World!!!
```
### (8). 使用autossh
> 上面通过ssh配置的反向代理的方式是不稳定的,这种ssh反向链接会因为超时而关闭,如果关闭了,那么外网连通内网的通道就无法维持了,为此我们需要另外的方法来提供稳定的ssh反向代理隧道工具(autossh).

```
# 1. Mac安装ssh
lixin-macbook:~ lixin$ brew install autossh
# 2. 通过autossh启动
# 参数说明:
#    -M   : 使用内网主机的55555端口监视SSH连接状态,连接出问题了会自动重连.
#    -N   : 不执行远程命令
#    -R   : 将公网主机的某个端口转发到本地指定机器的指定端口.
lixin-macbook:~ lixin$ autossh -M 55555 -NfR 8081:localhost:8080 root@47.119.169.76
# 3. 通过外网访问
lixin-macbook:~ lixin$ curl http://47.119.169.76:8081/hello
Hello World!!!
```
### (9). 总结
> 通过SSH反向隊道代理,可以实现内网和外网的穿透(通过访问外网的IP:PORT,透传到内网的IP:PORT).  