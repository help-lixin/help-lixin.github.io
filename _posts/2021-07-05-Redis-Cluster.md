---
layout: post
title: 'Redis Cluster搭建'
date: 2021-07-05
author: 李新
tags:  Redis 
---

### (1). 机器准备

|  机器IP         | port  |
|  ----          | ----  |
|  127.0.0.1     |  6380 |
|  127.0.0.1     |  6381 |
|  127.0.0.1     |  6382 |
|  127.0.0.1     |  6383 |
|  127.0.0.1     |  6384 |
|  127.0.0.1     |  6385 |

### (2). 源码编译安装
```
# 1. 安装依赖(linux下需要安装gcc,因为我是mac,所以,注释掉了)
# yum -y install gcc

# 2. 进入下载目录
lixin-macbook:~ lixin$ cd  ~/Downloads/

# 3. 下载redis
lixin-macbook:Downloads lixin$ wget https://download.redis.io/releases/redis-5.0.12.tar.gz

# 4. 解压redis
lixin-macbook:Downloads lixin$ tar -zxvf redis-5.0.12.tar.gz
lixin-macbook:Downloads lixin$ cd redis-5.0.12/

# 5. 编译源码
lixin-macbook:redis-5.0.12 lixin$ make && make install

#6. 运行redis服务端
lixin-macbook:redis-5.0.12 lixin$ ./src/redis-server ./redis.conf
7981:C 05 Jul 2021 16:29:27.851 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
7981:C 05 Jul 2021 16:29:27.851 # Redis version=5.0.12, bits=64, commit=00000000, modified=0, pid=7981, just started
7981:C 05 Jul 2021 16:29:27.851 # Configuration loaded
7981:M 05 Jul 2021 16:29:27.852 * Increased maximum number of open files to 10032 (it was originally set to 1024).
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 5.0.12 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 7981
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'
```
### (3). 创建工程目录
```
lixin-macbook:~ lixin$ cd ~/Developer/
lixin-macbook:Downloads lixin$ mkdir redis-cluster
lixin-macbook:Downloads lixin$ cd redis-cluster

# 1. 创建bin/conf/data/logs目录
lixin-macbook:redis-cluster lixin$ mkdir {bin,conf,data,logs}
#    创建数据目录
lixin-macbook:redis-cluster lixin$ mkdir data/{6380,6381,6382,6383,6384,6385}
#    创建日志目录
lixin-macbook:redis-cluster lixin$ mkdir logs/{6380,6381,6382,6383,6384,6385}

# 2. 拷贝编译过后的redis二进制文件到bin目录下
lixin-macbook:redis-cluster lixin$ cp ~/Downloads/redis-5.0.12/src/{redis-benchmark,redis-check-aof,redis-check-rdb,redis-cli,redis-sentinel,redis-server} ./bin/

# 3. 最终项目结构如下:
lixin-macbook:redis-cluster lixin$ tree
.
├── bin
│   ├── redis-benchmark
│   ├── redis-check-aof
│   ├── redis-check-rdb
│   ├── redis-cli
│   ├── redis-sentinel
│   └── redis-server
├── conf
├── data
│   ├── 6380
│   ├── 6381
│   ├── 6382
│   ├── 6383
│   ├── 6384
│   └── 6385
└── logs
    ├── 6380
    ├── 6381
    ├── 6382
    ├── 6383
    ├── 6384
    └── 6385
```
### (4). 创建配置文件
```
lixin-macbook:redis-cluster lixin$ cp ~/Downloads/redis-5.0.12/redis.conf  ./conf/redis-6380.conf

# ****************************************
# 注意:凡是我标有星号(*),代表需要变量替换的(6380/6381/6382/6383/6384/6385)
# ****************************************

# 1. 开启后台运行
   daemonize yes

# 2. 不能只绑定127.0.0.1,要允许其它机器访问
  # bind 127.0.0.1
  
# 3. 修改端口号(*)
  port 6380


# 4. 修改dir(*)
  dir /Users/lixin/Developer/redis-cluster/data/6380/

# 5. 开启集群模式
  cluster-enabled yes


# 6. 开启集群节点信息文件(*),这个文件是Redis集群节点每次发生更改时自动保留群集配置.
  cluster-config-file nodes-6380.conf

# 7. 打开集群超时
  cluster-node-timeout 15000

# 8. 关闭保护模式
  protected-mode no

# 9. 开启aof
  appendonly yes

# 10. 配置pid(*)
  pidfile /Users/lixin/Developer/redis-cluster/data/6380/redis_6380.pid

# 11. 配置logs(*)
  logfile /Users/lixin/Developer/redis-cluster/logs/6380/redis_6380.log

# 12. 设置登录密码
  requirepass 888888

# 13. 主从复制时的密码(因为在第12步有配Redis密码,所以这里也需要配置)
  masterauth 888888
```
### (5). 创建启动脚本
```
lixin-macbook:redis-cluster lixin$ cat start-all.sh
#!/bin/bash
/Users/lixin/Developer/redis-cluster/bin/redis-server /Users/lixin/Developer/redis-cluster/conf/redis-6380.conf
/Users/lixin/Developer/redis-cluster/bin/redis-server /Users/lixin/Developer/redis-cluster/conf/redis-6381.conf
/Users/lixin/Developer/redis-cluster/bin/redis-server /Users/lixin/Developer/redis-cluster/conf/redis-6382.conf
/Users/lixin/Developer/redis-cluster/bin/redis-server /Users/lixin/Developer/redis-cluster/conf/redis-6383.conf
/Users/lixin/Developer/redis-cluster/bin/redis-server /Users/lixin/Developer/redis-cluster/conf/redis-6384.conf
/Users/lixin/Developer/redis-cluster/bin/redis-server /Users/lixin/Developer/redis-cluster/conf/redis-6385.conf
```
### (6). 启动所有redis
```
lixin-macbook:redis-cluster lixin$ chmod +x start-all.sh
lixin-macbook:redis-cluster lixin$ ./start-all.sh
# 检查端口是否监听
lixin-macbook:redis-cluster lixin$ ps -ef|grep redis|grep -v grep
  501  2248     1   0  9:21下午 ??         0:00.39 /Users/lixin/Developer/redis-cluster/bin/redis-server *:6380 [cluster]
  501  2250     1   0  9:21下午 ??         0:00.37 /Users/lixin/Developer/redis-cluster/bin/redis-server *:6381 [cluster]
  501  2252     1   0  9:21下午 ??         0:00.40 /Users/lixin/Developer/redis-cluster/bin/redis-server *:6382 [cluster]
  501  2254     1   0  9:21下午 ??         0:00.38 /Users/lixin/Developer/redis-cluster/bin/redis-server *:6383 [cluster]
  501  2256     1   0  9:21下午 ??         0:00.39 /Users/lixin/Developer/redis-cluster/bin/redis-server *:6384 [cluster]
  501  2258     1   0  9:21下午 ??         0:00.37 /Users/lixin/Developer/redis-cluster/bin/redis-server *:6385 [cluster]
```
### (7). 集群初始化
```
# lixin-macbook:redis-cluster lixin$ ./bin/redis-cli --cluster help

#  --cluster-replicas 1  : 集群为1主1从
lixin-macbook:redis-cluster lixin$  ./bin/redis-cli -a 888888 --cluster create 127.0.0.1:6380 127.0.0.1:6381 127.0.0.1:6382 127.0.0.1:6383 127.0.0.1:6384 127.0.0.1:6385  --cluster-replicas 1 

Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383

# Slave与Master绑定
Adding replica 127.0.0.1:6384 to 127.0.0.1:6380
Adding replica 127.0.0.1:6385 to 127.0.0.1:6381
Adding replica 127.0.0.1:6383 to 127.0.0.1:6382
>>> Trying to optimize slaves allocation for anti-affinity
[WARNING] Some slaves are in the same host as their master

# Master信息
M: d4fb72a249a42ecfce9f5e6d90eab917ddf20c22 127.0.0.1:6380
   slots:[0-5460] (5461 slots) master
M: d36cf78e5481e488bc50dd31960044cc1e69522f 127.0.0.1:6381
   slots:[5461-10922] (5462 slots) master
M: 8a9bf55091d15fd77de0bf7d4db05e09a5b682db 127.0.0.1:6382
   slots:[10923-16383] (5461 slots) master
S: bf09356f37d20288d1a8cb5754b0f5b6afddfdae 127.0.0.1:6383
   replicates d36cf78e5481e488bc50dd31960044cc1e69522f
S: 894425fde65f7da875d4d078f41e01381e52c6d9 127.0.0.1:6384
   replicates 8a9bf55091d15fd77de0bf7d4db05e09a5b682db
S: edaeeffe57750afe67957f9ebbbcb0d919e9baea 127.0.0.1:6385
   replicates d4fb72a249a42ecfce9f5e6d90eab917ddf20c22
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
>>> Performing Cluster Check (using node 127.0.0.1:6380)
M: d4fb72a249a42ecfce9f5e6d90eab917ddf20c22 127.0.0.1:6380
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: bf09356f37d20288d1a8cb5754b0f5b6afddfdae 127.0.0.1:6383
   slots: (0 slots) slave
   replicates d36cf78e5481e488bc50dd31960044cc1e69522f
S: 894425fde65f7da875d4d078f41e01381e52c6d9 127.0.0.1:6384
   slots: (0 slots) slave
   replicates 8a9bf55091d15fd77de0bf7d4db05e09a5b682db
S: edaeeffe57750afe67957f9ebbbcb0d919e9baea 127.0.0.1:6385
   slots: (0 slots) slave
   replicates d4fb72a249a42ecfce9f5e6d90eab917ddf20c22
M: d36cf78e5481e488bc50dd31960044cc1e69522f 127.0.0.1:6381
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
M: 8a9bf55091d15fd77de0bf7d4db05e09a5b682db 127.0.0.1:6382
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```
### (8). client连接测试
```
# -c   : 智能客户端,以集群的模式连接上Redis.
lixin-macbook:redis-cluster lixin$ ./bin/redis-cli -c  -a 888888 -p 6380

# 测试,设置数据
127.0.0.1:6380> set a a
-> Redirected to slot [15495] located at 127.0.0.1:6382
OK

# 测试,获取数据
127.0.0.1:6382> get a
"a"
```
### (9). Redis集群命令
```
# 1. 查看key的hash slot
127.0.0.1:6382> CLUSTER KEYSLOT a
(integer) 15495

# 2. 查看所有的slot
127.0.0.1:6380> CLUSTER SLOTS
1) 1) (integer) 5461
   2) (integer) 10922
   3) 1) "127.0.0.1"
      2) (integer) 6381(M)
      3) "d36cf78e5481e488bc50dd31960044cc1e69522f"
   4) 1) "127.0.0.1"
      2) (integer) 6383(S)
      3) "bf09356f37d20288d1a8cb5754b0f5b6afddfdae"
2) 1) (integer) 10923
   2) (integer) 16383
   3) 1) "127.0.0.1"
      2) (integer) 6382(M)
      3) "8a9bf55091d15fd77de0bf7d4db05e09a5b682db"
   4) 1) "127.0.0.1"
      2) (integer) 6384(S)
      3) "894425fde65f7da875d4d078f41e01381e52c6d9"
3) 1) (integer) 0
   2) (integer) 5460
   3) 1) "127.0.0.1"
      2) (integer) 6380(M)
      3) "d4fb72a249a42ecfce9f5e6d90eab917ddf20c22"
   4) 1) "127.0.0.1"
      2) (integer) 6385(S)
      3) "edaeeffe57750afe67957f9ebbbcb0d919e9baea"

# 3. 查看集群信息
127.0.0.1:6380> CLUSTER INFO
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:6
cluster_my_epoch:1
cluster_stats_messages_ping_sent:907
cluster_stats_messages_pong_sent:922
cluster_stats_messages_sent:1829
cluster_stats_messages_ping_received:917
cluster_stats_messages_pong_received:907
cluster_stats_messages_meet_received:5
cluster_stats_messages_received:1829
```
### (10). 总结
> Redis5.X已经不再需要Ruby,看来,它已经慢慢的把生态圈开始完善了.  