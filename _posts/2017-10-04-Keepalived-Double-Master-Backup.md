---
layout: post
title: 'Keepalived 双机热备(三)'
date: 2017-10-04
author: 李新
tags: Keepalived
---

### (1). 前言
> 在前面,运用了:Keepalived实现了高可用,但是,有个缺点就是:在同时刻只有一台机器工作,另一台备份机器在主机器不出现故障的时候,永远处于浪费状态,对于服务器不多的网站,该方案不经济实惠.   

### (2). 机器准备

|  机器名称   |        ip       |    vip                        |
|  ----      |      ----       |   ----                        |
| app_100    | 10.211.55.100   |   10.211.55.88                |
| app_101    | 10.211.55.101   |   10.211.55.89                |

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
### (9). 配置app-100(/etc/keepalived/keepalived.conf)
```
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
    state MASTER            # 备份服务器上将 MASTER 改为 BACKUP
    interface eth0
    virtual_router_id 100   # 主,备机的virtual_router_id必须相同 
    master_src_ip 10.211.55.100
    nopreempt
    priority 100            # 主,备机取不同的优先级,主机值较大,备份机值较小
    advert_int 1

    track_script {
       check-nginx
    }

    virtual_ipaddress {
       10.211.55.88
    }
}


vrrp_instance VI_2 {
    state BACKUP            # 备份服务器上将 MASTER 改为 BACKUP
    interface eth0
    virtual_router_id 200   # 主,备机的virtual_router_id必须相同 
    master_src_ip 10.211.55.100
    nopreempt
    priority 90            # 主,备机取不同的优先级,主机值较大,备份机值较小
    advert_int 1

    track_script {
       check-nginx
    }

    virtual_ipaddress {
       10.211.55.89
    }
}
```
### (10). 配置从节点app-101(/etc/keepalived/keepalived.conf)
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

vrrp_instance VI_2 {
    state MASTER            # 备份服务器上将 MASTER 改为 BACKUP
    interface eth0
    virtual_router_id 200   # 主,备机的virtual_router_id必须相同 
    master_src_ip 10.211.55.101
    nopreempt
    priority 100            # 主,备机取不同的优先级,主机值较大,备份机值较小
    advert_int 1

    track_script {
       check-nginx
    }

    virtual_ipaddress {
       10.211.55.89
    }
}
```
### (11). 创建检查脚本(/home/check.sh)
```
# 尝试启动nginx,如果,启动失败的情况下,杀掉keepalived进程(让出VIP资源) 
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
### (12). 测试结果
```
# 1. app-100启动nginx和keepalived
[root@app-100 ~]# /usr/local/nginx/sbin/nginx
[root@app-100 ~]# keepalived

# 2. app-101启动nginx和keepalived
[root@app-101 ~]# /usr/local/nginx/sbin/nginx
[root@app-101 ~]# keepalived


# 3. 在主机没有出现故障的情况下,我们看下(app-100)VIP的情况
[root@app-100 ~]# ip addr
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:1c:42:2e:64:5e brd ff:ff:ff:ff:ff:ff
    inet 10.211.55.100/24 brd 10.211.55.255 scope global noprefixroute eth0
       valid_lft forever preferred_lft forever
    inet 10.211.55.88/32 scope global eth0


# 4. 在主机没有出现故障的情况下,我们看下(app-101)VIP的情况
[root@app-101 ~]# ip addr
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:1c:42:b2:80:e9 brd ff:ff:ff:ff:ff:ff
    inet 10.211.55.101/24 brd 10.211.55.255 scope global noprefixroute eth0
       valid_lft forever preferred_lft forever
    inet 10.211.55.89/32 scope global eth0
       valid_lft forever preferred_lft forever

# 5. 89与app-101主机绑定,推断返回结果应该是:app-101
lixin-macbook:~ lixin$ curl http://10.211.55.89
<h1>Welcome to nginx app-101</h1>

# 6. 88与app-100主机绑定,推断返回结果应该是:app-100
lixin-macbook:~ lixin$ curl http://10.211.55.88
<h1>Welcome to nginx app-100</h1>

# 7. 模拟(app-100)机器出现故障
[root@app-100 ~]# killall nginx
[root@app-100 ~]# killall keepalived

# *************************************************************************
# 8. 检查下app-100的ip信息(10.211.55.88的IP已经飘移走了)
# *************************************************************************
[root@app-100 ~]# ip addr
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:1c:42:2e:64:5e brd ff:ff:ff:ff:ff:ff
    inet 10.211.55.100/24 brd 10.211.55.255 scope global noprefixroute eth0
       valid_lft forever preferred_lft forever
    inet6 fdb2:2c26:f4e4:0:21c:42ff:fe2e:645e/64 scope global noprefixroute dynamic
       valid_lft 2591893sec preferred_lft 604693sec
    inet6 fe80::21c:42ff:fe2e:645e/64 scope link noprefixroute
       valid_lft forever preferred_lft forever

# *************************************************************************
# 9. 检查下app-101的ip信息,10.211.55.88/10.211.55.89都绑定在了eth0网卡上了
# *************************************************************************
[root@app-101 ~]# ip addr
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:1c:42:b2:80:e9 brd ff:ff:ff:ff:ff:ff
    inet 10.211.55.101/24 brd 10.211.55.255 scope global noprefixroute eth0
       valid_lft forever preferred_lft forever
    inet 10.211.55.89/32 scope global eth0
       valid_lft forever preferred_lft forever
    inet 10.211.55.88/32 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fdb2:2c26:f4e4:0:21c:42ff:feb2:80e9/64 scope global noprefixroute dynamic
       valid_lft 2591830sec preferred_lft 604630sec
    inet6 fe80::21c:42ff:feb2:80e9/64 scope link noprefixroute
       valid_lft forever preferred_lft forever


# 10. 此时,不论访问哪个VIP,都会返回:app-101
lixin-macbook:~ lixin$ curl http://10.211.55.89
<h1>Welcome to nginx app-101</h1>
lixin-macbook:~ lixin$ curl http://10.211.55.88
<h1>Welcome to nginx app-101</h1>

# *************************************************************************
# 11. app-100重新开启:keepalived,需要验证IP:10.211.55.88会绑定到:app-100的机器上
# *************************************************************************
[root@app-100 ~]# /usr/local/nginx/sbin/nginx
[root@app-100 ~]# keepalived

# *************************************************************************
# 12. 如预期所想,IP会飘移回来.
# *************************************************************************
[root@app-100 ~]# ip addr
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:1c:42:2e:64:5e brd ff:ff:ff:ff:ff:ff
    inet 10.211.55.100/24 brd 10.211.55.255 scope global noprefixroute eth0
       valid_lft forever preferred_lft forever
    inet 10.211.55.88/32 scope global eth0


[root@app-101 ~]# ip addr
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:1c:42:b2:80:e9 brd ff:ff:ff:ff:ff:ff
    inet 10.211.55.101/24 brd 10.211.55.255 scope global noprefixroute eth0
       valid_lft forever preferred_lft forever
    inet 10.211.55.89/32 scope global eth0
       valid_lft forever preferred_lft forever


# 13.测试关闭nginx,验证:keepalived会重新启动ngingx
[root@app-100 ~]# ip addr
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:1c:42:2e:64:5e brd ff:ff:ff:ff:ff:ff
    inet 10.211.55.100/24 brd 10.211.55.255 scope global noprefixroute eth0
       valid_lft forever preferred_lft forever
    inet 10.211.55.88/32 scope global eth0
       valid_lft forever preferred_lft forever
# *************************************************************************	   
## kill2次,你会发现PID是变了,代表确实是有在:kill,只是keepalived很快执行脚本,帮我们启动nginx
# *************************************************************************
[root@app-100 ~]# killall nginx
[root@app-100 ~]# ps -ef|grep nginx|grep -v grep  
root      3998     1  0 12:29 ?        00:00:00 nginx: master process /usr/local/nginx/sbin/nginx
nobody    4000  3998  0 12:29 ?        00:00:00 nginx: worker process

[root@app-100 ~]# killall nginx
[root@app-100 ~]# ps -ef|grep nginx|grep -v grep 
root      4062     1  0 12:29 ?        00:00:00 nginx: master process /usr/local/nginx/sbin/nginx
nobody    4064  4062  0 12:29 ?        00:00:00 nginx: worker process
```
### (14). 总结
> 所谓的高可用,就是解决虚拟IP的飘移问题,其实,是会存在脑裂的问题来着的.我们自己也可以基于ZK,实现这么一套VIP飘移.  