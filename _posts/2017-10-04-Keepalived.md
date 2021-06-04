---
layout: post
title: 'Keepalived 主备介绍'
date: 2017-10-04
author: 李新
tags: Keepalived
---

### (1). 前言
> Keepalived起初是为LVS设计的,专门用来监控集群系统中各个服务节点的状态,它根据TCP/IP参考模型的第三、第四层、第五层交换机制检测每个服务节点的状态,如果某个服务器节点出现异常,或者工作出现故障,Keepalived将检测到,并将出现的故障的服务器节点从集群系统中剔除,这些工作全部是自动完成的,不需要人工干涉,需要人工完成的只是修复出现故障的服务节点.

### (2). 机器准备

|  机器名称   |                 |    vip           |
|  ----      |      ----       |   ----           |
| app_100    | 10.211.55.100   |   10.211.55.88   |
| app_101    | 10.211.55.101   |   10.211.55.88   |

### (3). 安装准备
```
# 关闭防火墙
# systemctl stop firewalld.service

# 禁用防火墙
# systemctl disable firewalld.service
```
### (4). Nginx安装
```
# yum -y install gcc gcc-c++  pcre pcre-devel zlib zlib-devel openssl openssl-devel
# mkdir -p /opt/soft
# cd /opt/soft/
# wget http://nginx.org/download/nginx-1.18.0.tar.gz
# tar -zxvf nginx-1.18.0.tar.gz
# cd nginx-1.18.0/
# ./configure --prefix=/usr/local/nginx
# make && make install
# ll /usr/local/nginx/
drwxr-xr-x. 2 root root 4096 Jun  4 14:52 conf
drwxr-xr-x. 2 root root 4096 Jun  4 14:52 html
drwxr-xr-x. 2 root root 4096 Jun  4 14:52 logs
drwxr-xr-x. 2 root root 4096 Jun  4 14:52 sbin

# 启动nginx
# /usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
```
### (5). app-100修改静态页面
```
[root@app-100 ~]# sed -i  's/nginx!/nginx app-100/' /usr/local/nginx/html/index.html
```
### (6). app-101修改静态页面
```
[root@app-101 ~]# sed -i  's/nginx!/nginx app-101/' /usr/local/nginx/html/index.html
```
### (7). Keepalived安装
```
# 1. 下载keepalived
# wget https://www.keepalived.org/software/keepalived-1.2.18.tar.gz

# 2. 解压
# tar -zxvf keepalived-1.2.18.tar.gz

# 3. 编译并安装
# cd keepalived-1.2.18/
# ./configure --prefix=/usr/local/keepalived
# make && make install
```
### (8). Keepalived初始化
```
# 1. 创建Keepalived配置目录
# mkdir /etc/keepalived

# 2. 拷贝配置
# cp /usr/local/keepalived/etc/keepalived/keepalived.conf  /etc/keepalived/

# 3. 配置开机启动脚本
# cp /usr/local/keepalived/etc/rc.d/init.d/keepalived  /etc/init.d/
# cp /usr/local/keepalived/etc/sysconfig/keepalived  /etc/sysconfig/
# ln -s /usr/local/keepalived/sbin/keepalived  /usr/sbin/
# chkconfig keepalived on
```
### (9). 配置app-100
```
# vi /etc/keepalived/keepalived.conf
global_defs {
    router_id app-100
}

vrrp_script check-nginx
{
    script "/home/check.sh"
    interval 2
    weight -20
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 100   # 主、备机的virtual_router_id必须相同 
    master_src_ip 10.211.55.100
    nopreempt
    priority 100            # 主,备机的权重要求不一样(备机一般比主要的权重要小)
    advert_int 1

    track_script {
       check-nginx
    }

    virtual_ipaddress {
       10.211.55.88
    }
}
```
### (10). 配置从节点(app-101)
```
global_defs {
    router_id app-101
}

vrrp_script check-nginx
{
    script "/home/check.sh"
    interval 2
    weight -20
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 100
    master_src_ip 10.211.55.101
    nopreempt
    priority 80
    advert_int 1

    track_script {
       check-nginx
    }

    virtual_ipaddress {
       10.211.55.88
    }
}
```
### (11). 创建检查脚本
```
# vi /home/check.sh

# 尝试启动nginx,如果,启动失败的情况下,杀掉keepalived进程
#!/bin/bash
nginx_count=`ps -ef|grep nginx|grep -v grep|wc -l`
if [ $nginx_count -eq 0 ];then
    /usr/local/nginx/sbin/nginx
    sleep 3
    nginx_count=`ps -ef|grep nginx|grep -v grep|wc -l`

    if [ $nginx_count -eq 0 ];then
        killall keepalived
    fi
fi
```
### (12). 测试
```
# 1. 启动keepalived
[root@app-100 ~]# keepalived
[root@app-101 ~]# keepalived

# 2. 查看(app-100)IP信息(此时,Master绑定了一个虚拟IP)
[root@app-100 ~]# ip addr
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:1c:42:2e:64:5e brd ff:ff:ff:ff:ff:ff
    inet 10.211.55.100/24 brd 10.211.55.255 scope global noprefixroute eth0
       valid_lft forever preferred_lft forever
    inet 10.211.55.88/32 scope global eth0
       valid_lft forever preferred_lft forever

# 2. 查看(app-101)IP信息(此时,BACKUP没有绑定任何的虚拟IP)
[root@app-101 ~]# ip addr
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:1c:42:b2:80:e9 brd ff:ff:ff:ff:ff:ff
    inet 10.211.55.101/24 brd 10.211.55.255 scope global noprefixroute eth0
       valid_lft forever preferred_lft forever
    inet6 fdb2:2c26:f4e4:0:21c:42ff:feb2:80e9/64 scope global noprefixroute dynamic
       valid_lft 2591814sec preferred_lft 604614sec
    inet6 fe80::21c:42ff:feb2:80e9/64 scope link noprefixroute
       valid_lft forever preferred_lft forever

# 3. 测试访问
lixin-macbook:~ lixin$ curl http://10.211.55.88
<h1>Welcome to nginx app-100</h1>

# 4. 关闭主节点,验证VIP是否会飘移,服务是否能正常访问
[root@app-100 ~]# killall keepalived

# 5. 经验证,VIP在MASTER节点已经撤销(那么BACKUP是否会绑定VIP呢?).
[root@app-100 ~]# ip addr
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:1c:42:2e:64:5e brd ff:ff:ff:ff:ff:ff
    inet 10.211.55.100/24 brd 10.211.55.255 scope global noprefixroute eth0
       valid_lft forever preferred_lft forever
    inet6 fdb2:2c26:f4e4:0:21c:42ff:fe2e:645e/64 scope global noprefixroute dynamic
       valid_lft 2591710sec preferred_lft 604510sec
    inet6 fe80::21c:42ff:fe2e:645e/64 scope link noprefixroute
       valid_lft forever preferred_lft forever

# 6. 查看BACKUP是否绑定VIP,确实有绑定VIP来着的
[root@app-101 ~]# ip addr
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:1c:42:b2:80:e9 brd ff:ff:ff:ff:ff:ff
    inet 10.211.55.101/24 brd 10.211.55.255 scope global noprefixroute eth0
       valid_lft forever preferred_lft forever
    inet 10.211.55.88/32 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fdb2:2c26:f4e4:0:21c:42ff:feb2:80e9/64 scope global noprefixroute dynamic
       valid_lft 2591668sec preferred_lft 604468sec
    inet6 fe80::21c:42ff:feb2:80e9/64 scope link noprefixroute
       valid_lft forever preferred_lft forever

# 7. 测试重新访问
lixin-macbook:~ lixin$ curl http://10.211.55.88
<h1>Welcome to nginx app-101</h1>


# 8. 重新启动主节点(MASTER会重新抢过VIP绑定在自己的网卡上)
[root@app-100 ~]# keepalived
[root@app-100 ~]# ip addr
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:1c:42:2e:64:5e brd ff:ff:ff:ff:ff:ff
    inet 10.211.55.100/24 brd 10.211.55.255 scope global noprefixroute eth0
       valid_lft forever preferred_lft forever
    inet 10.211.55.88/32 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fdb2:2c26:f4e4:0:21c:42ff:fe2e:645e/64 scope global noprefixroute dynamic
       valid_lft 2591581sec preferred_lft 604381sec
    inet6 fe80::21c:42ff:fe2e:645e/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
# BACKUP会撤销VIP的绑定
[root@app-101 ~]# ip addr
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:1c:42:b2:80:e9 brd ff:ff:ff:ff:ff:ff
    inet 10.211.55.101/24 brd 10.211.55.255 scope global noprefixroute eth0
       valid_lft forever preferred_lft forever
    inet6 fdb2:2c26:f4e4:0:21c:42ff:feb2:80e9/64 scope global noprefixroute dynamic
       valid_lft 2591576sec preferred_lft 604376sec
    inet6 fe80::21c:42ff:feb2:80e9/64 scope link noprefixroute
       valid_lft forever preferred_lft forever

# 9. 测试访问
lixin-macbook:~ lixin$ curl http://10.211.55.88
<h1>Welcome to nginx app-100</h1>
```
### (14). 总结
> 通过Keepalived即可实现:VIP的飘移功能,但,有个缺陷就是:一主一备,备机资源是闲置的(两台机器互为双主,将另开一篇来玩).  