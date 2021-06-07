---
layout: post
title: 'Linux 常用命令'
date: 2017-06-04
author: 李新
tags: Linux
---

### (1). sed命令
> sed命令主要是用脚本来编辑文本内容

```
# sed 语法
# s        : 替换
# g        : 全局替换
# pattern  : 模式(要扫描的内容)
# -i       : 直接修改读取的文件内容,而不是输出到终端.
# sed 's/pattern/replace content/g' inputFileName > outputFileName
sed [OPTION]... {script-only-if-no-other-script} [input-file]...

# 演示文本内容
[root@test ~]# cat demo.txt
1,张三,25
2,李四,26
3,王五,27
4,赵六,28

# 把所有的逗号替换成竖线.
[root@test ~]# sed 's/,/|/g' demo.txt
1|张三|25
2|李四|26
3|王五|27
4|赵六|28
```

### (2). awk命令
> awk是一种编程语言,可以用来处理数据和生成报告. 

```
# awk语法
# options   :  参数
# pattern   :  模式(可以理解成条件)
# action    :  动作(是由大括号里面的一条或者多条语句组成,语句之间使用分号隔开).
awk [options] 'pattern {action}' file

# 把一行记录,按照冒号(:),进行split,然后,根据用$来取下标对应的值
# NR       : 行号记录数
# $0       : $0代表取整行.
# $1 ~ $N  : 取N下标的值.
# $NF      : 下标中最后一个值.
# RS       : 指定行分隔符,默认一行的分隔符是回车.
# FS       : 指定字段分隔符
# OFS      : 指定输出字段分隔符
[root@test ~]# awk -F ":" 'NR >= 2 && NR <=5 {print NR,$1,$NF}'  /etc/passwd

# 以冒号(:)为分隔符,并打印每一行的数据.
[root@test ~]# awk 'BEGIN{RS=":"} {print NR,$0}' /etc/passwd
# 以冒号(:)为分隔符,在打印数据时,数据之间用逗号(,)进行分隔
[root@test ~]# awk 'BEGIN{FS=":";OFS=","} {print NR,$1,$2,$3,$4,$5,$6} END { print "end!" }' /etc/passwd

# kill掉mongodb的进程.
[root@test ~]# kill ` ps -ef|grep  mongodb | grep -v grep | awk '{print $2}' `
```

### (3). 增加虚拟IP(VIP)
> 为后续Keepalived打个基础.  

```
# 1. 查看机器上现有的网卡信息
[root@test ~]# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.211.55.100  netmask 255.255.255.0  broadcast 10.211.55.255

# 2. 为网卡添加一个虚拟IP
[root@test ~]# ifconfig eth0:1 10.211.55.111 netmask  255.255.255.0 up  

# 3. 再次查看网卡信息
[root@test ~]# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.211.55.100  netmask 255.255.255.0  broadcast 10.211.55.255

eth0:1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.211.55.111  netmask 255.255.255.0  broadcast 10.211.55.255
        ether 00:1c:42:e6:6f:3f  txqueuelen 1000  (Ethernet)

# 4. 测试
lixin-macbook:~ lixin$ ping 10.211.55.111
PING 10.211.55.111 (10.211.55.111): 56 data bytes
64 bytes from 10.211.55.111: icmp_seq=0 ttl=64 time=0.331 ms
64 bytes from 10.211.55.111: icmp_seq=1 ttl=64 time=1.784 ms
```

### (4). netstat命令详解
```
# netstat参数信息:
-a (all) 显示所有选项(LISTEN/ESTABLISHED/TIME_WAIT/CLOSED).
-t (tcp) 仅显示tcp相关选项.
-u (udp) 仅显示udp相关选项.
-n 拒绝显示别名,能显示数字的全部转化成数字.
-l 仅列出有在Listen(监听)的服务状态.

-p 显示建立相关链接的程序名
-r 显示路由信息,路由表
-e 显示扩展信息,例如uid等
-s 按各个协议进行统计
-c 每隔一个固定时间,执行该netstat命令。

# State状态说明:
# SYN_SENT     : Client发送connect请求时状态
# LISTEN       : 侦听来自远方的TCP端口的连接请求
# SYN_RCVD     : Server向Client发送同步接受数据.
# ESTABLISHED  : 代表一个打开的连接(连接已建立)

# FIN_WAIT_1    : Client发送连接关闭请求.
# CLOSE_WAIT    : Server接受到client发送过来的连接中断请求
# FIN_WAIT_2    : Server向Client发送连接关闭请求
# LAST_ACK      : Server最后确认close 
# TIME_WAIT     : 等待client关闭. 

[root@tomcat-1 ~]# netstat -atlnp
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:111             0.0.0.0:*               LISTEN      1003/rpcbind
tcp        0      0 192.168.122.1:53        0.0.0.0:*               LISTEN      2173/dnsmasq
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      1755/sshd
tcp        0      0 127.0.0.1:631           0.0.0.0:*               LISTEN      1753/cupsd
tcp        0      0 127.0.0.1:25            0.0.0.0:*               LISTEN      2237/master
tcp        0      0 10.37.129.3:56756       10.37.129.4:22          ESTABLISHED 2817/ssh
tcp        0      0 10.211.55.100:22        10.211.55.2:49657       ESTABLISHED 2699/sshd: root@pts
tcp        0      0 10.211.55.100:22        10.211.55.2:49658       ESTABLISHED 2748/sshd: root@pts
tcp        0      0 10.211.55.100:22        10.211.55.2:49698       ESTABLISHED 3238/sshd: root@pts
tcp6       0      0 :::111                  :::*                    LISTEN      1003/rpcbind
tcp6       0      0 :::22                   :::*                    LISTEN      1755/sshd
tcp6       0      0 ::1:631                 :::*                    LISTEN      1753/cupsd
tcp6       0      0 ::1:25                  :::*                    LISTEN      2237/master
```
### (5). tcpdump
```
#  -i   : 监听哪个端口
#  -s   : 数据包大小
#  -w   : 监听数据保存到文件
#  -r   : 读取数据
[root@tomcat-1 ~]# tcpdump -i eth0 -s 0 -w a.cap

# 以一个http的请求为案例
[root@tomcat-1 ~]# tcpdump -i eth0 'port 80'
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes

# 14:42:46.068855                               : 时间带有精确到微妙
# 10.211.55.2.50703 > tomcat-1.http             : 表示通信的方向(client->10.211.55.2.50703 / server->tomcat-1.http)

# Flags详解
# [SEW]                                         : 连接请求报文
# [S.E]                                         : 确认连接报文
# [.]                                           : ACK确认包
# [P.]                                          : 数据报文
# [F.]                                          : 释放报文


# *******************************************************************************************************************************
# 1. 握手包
# client->10.211.55.2.50703
# server->tomcat-1.http(10.211.55.100)
# *******************************************************************************************************************************
14:42:46.068855 IP 10.211.55.2.50703 > tomcat-1.http: Flags [SEW], seq 3164881139, win 65535, options [mss 1460,nop,wscale 6,nop,nop,TS val 540802254 ecr 0,sackOK,eol], length 0
14:42:46.068955 IP tomcat-1.http > 10.211.55.2.50703: Flags [S.E], seq 999336526, ack 3164881140, win 28960, options [mss 1460,sackOK,TS val 7014837 ecr 540802254,nop,wscale 7], length 0
14:42:46.069118 IP 10.211.55.2.50703 > tomcat-1.http: Flags [.], ack 1, win 2058, options [nop,nop,TS val 540802254 ecr 7014837], length 0

# 2. 发送(数据报文)http://10.211.55.100/请求,内容长度为:78
14:42:46.069191 IP 10.211.55.2.50703 > tomcat-1.http: Flags [P.], seq 1:78, ack 1, win 2058, options [nop,nop,TS val 540802254 ecr 7014837], length 77: HTTP: GET / HTTP/1.1
# 3. 服务端,向Client发送(确认报文)已经接受到78字节的数据
14:42:46.069215 IP tomcat-1.http > 10.211.55.2.50703: Flags [.], ack 78, win 227, options [nop,nop,TS val 7014838 ecr 540802254], length 0

# 4. 服务端,向Client发送(数据报文)以及内容(239字节)
14:42:46.071085 IP tomcat-1.http > 10.211.55.2.50703: Flags [P.], seq 1:239, ack 78, win 227, options [nop,nop,TS val 7014839 ecr 540802254], length 238: HTTP: HTTP/1.1 200 OK
# 5. Client回复Server(确认报文),收到:239个字节
14:42:46.071252 IP 10.211.55.2.50703 > tomcat-1.http: Flags [.], ack 239, win 2055, options [nop,nop,TS val 540802256 ecr 7014839], length 0

# 5. 服务端,向Client发送(数据报文)以及内容(612字节)
14:42:46.071317 IP tomcat-1.http > 10.211.55.2.50703: Flags [P.], seq 239:851, ack 78, win 227, options [nop,nop,TS val 7014840 ecr 540802256], length 612: HTTP
# 6. Client,向Server回复(确认报文),总共收到:851字节(239+612)
14:42:46.071686 IP 10.211.55.2.50703 > tomcat-1.http: Flags [.], ack 851, win 2045, options [nop,nop,TS val 540802256 ecr 7014840], length 0

# *******************************************************************************************************************************
# 7. 断开请求,我不太理解,为什么是3次?不是4次?
# *******************************************************************************************************************************
14:42:46.071740 IP 10.211.55.2.50703 > tomcat-1.http: Flags [F.], seq 78, ack 851, win 2048, options [nop,nop,TS val 540802256 ecr 7014840], length 0
14:42:46.071796 IP tomcat-1.http > 10.211.55.2.50703: Flags [F.], seq 851, ack 79, win 227, options [nop,nop,TS val 7014840 ecr 540802256], length 0
14:42:46.071905 IP 10.211.55.2.50703 > tomcat-1.http: Flags [.], ack 852, win 2048, options [nop,nop,TS val 540802256 ecr 7014840], length 0
```
### (6). vim常用快捷键
```
^       : 光标移动到行首
$       : 光标移动到行尾
1,17d   : 删除1-17行
yy      : 复制当前行
p       : 粘贴复制内容
dd      : 删除当前行
```