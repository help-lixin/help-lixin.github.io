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
### (4). vim常用快捷键
```
^       : 光标移动到行首
$       : 光标移动到行尾
1,17d   : 删除1-17行
yy      : 复制当前行
p       : 粘贴复制内容
dd      : 删除当前行
```
