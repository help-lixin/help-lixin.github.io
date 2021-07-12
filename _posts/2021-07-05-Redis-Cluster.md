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

# --cluster-replicas 表示有一个主有几个slave
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
# -c   : 以集群的模式连接上Redis.
lixin-macbook:redis-cluster lixin$ ./bin/redis-cli -c  -a 888888 -p 6380

# 测试,设置数据
127.0.0.1:6380> set a a
-> Redirected to slot [15495] located at 127.0.0.1:6382
OK

# 测试,获取数据
127.0.0.1:6382> get a
"a"
```
### (9). redis目录结构
```
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
│   ├── redis-6380.conf
│   ├── redis-6381.conf
│   ├── redis-6382.conf
│   ├── redis-6383.conf
│   ├── redis-6384.conf
│   └── redis-6385.conf
├── data
│   ├── 6380
│   │   ├── appendonly.aof
│   │   ├── dump.rdb
│   │   ├── nodes-6380.conf
│   │   └── redis_6380.pid
│   ├── 6381
│   │   ├── appendonly.aof
│   │   ├── dump.rdb
│   │   ├── nodes-6381.conf
│   │   └── redis_6381.pid
│   ├── 6382
│   │   ├── appendonly.aof
│   │   ├── dump.rdb
│   │   ├── nodes-6382.conf
│   │   └── redis_6382.pid
│   ├── 6383
│   │   ├── appendonly.aof
│   │   ├── dump.rdb
│   │   ├── nodes-6383.conf
│   │   └── redis_6383.pid
│   ├── 6384
│   │   ├── appendonly.aof
│   │   ├── dump.rdb
│   │   ├── nodes-6384.conf
│   │   └── redis_6384.pid
│   └── 6385
│       ├── appendonly.aof
│       ├── dump.rdb
│       ├── nodes-6385.conf
│       └── redis_6385.pid
├── logs
│   ├── 6380
│   │   └── redis_6380.log
│   ├── 6381
│   │   └── redis_6381.log
│   ├── 6382
│   │   └── redis_6382.log
│   ├── 6383
│   │   └── redis_6383.log
│   ├── 6384
│   │   └── redis_6384.log
│   └── 6385
│       └── redis_6385.log
└── start-all.sh
```
### (10). Redis集群命令
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
### (11). 模拟一组Redis(Master和Slave)出现故障
> 结论:   
> 在Redis集群有Master和Slave,如果,Master和Slave都下线后,整个集群是不能对外提供服务的.   

```
# 1. 查看集群现在的状态
127.0.0.1:6380> CLUSTER SLOTS
1) 1) (integer) 10923
   2) (integer) 16383
   3) 1) "127.0.0.1"
      2) (integer) 6382
      3) "8a9bf55091d15fd77de0bf7d4db05e09a5b682db"
   4) 1) "127.0.0.1"
      2) (integer) 6384
      3) "894425fde65f7da875d4d078f41e01381e52c6d9"
2) 1) (integer) 5461
   2) (integer) 10922
   3) 1) "127.0.0.1"
      2) (integer) 6381
      3) "d36cf78e5481e488bc50dd31960044cc1e69522f"
   4) 1) "127.0.0.1"
      2) (integer) 6383
      3) "bf09356f37d20288d1a8cb5754b0f5b6afddfdae"
3) 1) (integer) 0
   2) (integer) 5460
   3) 1) "127.0.0.1"
      2) (integer) 6380
      3) "d4fb72a249a42ecfce9f5e6d90eab917ddf20c22"
   4) 1) "127.0.0.1"
      2) (integer) 6385
      3) "edaeeffe57750afe67957f9ebbbcb0d919e9baea"

# 2. 添加一条数据,路由是在:127.0.0.1:6382.
#    kill掉6382和6384两个端口对应的服务,如果不关闭6384的话,6384会过一段时间,提升为master节点.整个集群,仍然是可以对外提供服务的.
127.0.0.1:6380> set a a
-> Redirected to slot [15495] located at 127.0.0.1:6382
OK

# 3. 检查6382和6384
lixin-macbook:~ lixin$ ps -ef|grep redis|grep 6382
  501   755     1   0  9:46下午 ??         0:01.15 /Users/lixin/Developer/redis-cluster/bin/redis-server *:6382 [cluster]
lixin-macbook:~ lixin$ ps -ef|grep redis|grep 6384
  501   759     1   0  9:46下午 ??         0:01.20 /Users/lixin/Developer/redis-cluster/bin/redis-server *:6384 [cluster]

# 4. kill掉对应的进程
lixin-macbook:~ lixin$ kill $(ps -ef|grep redis|grep 6382|grep -v grep|awk '{print $2}')
lixin-macbook:~ lixin$ kill $(ps -ef|grep redis|grep 6384|grep -v grep|awk '{print $2}')

# 5. 再次验证,是否kill结束
lixin-macbook:~ lixin$ ps -ef|grep redis|grep 6382
lixin-macbook:~ lixin$ ps -ef|grep redis|grep 6384

# 6. 验证集群是否能正常的插入数据,修改数据,删除数据
lixin-macbook:redis-cluster lixin$ ./bin/redis-cli -c -a 888888 -p 6380
# 6.1 获取已经保存的值,提示集群已经关闭
127.0.0.1:6380> get a
(error) CLUSTERDOWN The cluster is down

# 6.2 查看key(b)将会保存在哪个solt(3300会保存在6380服和器上)
127.0.0.1:6380> CLUSTER KEYSLOT b
(integer) 3300

# 6.3 集群中,只要有一组(一组M和S)机器出现故障,那么,整个集群是不可以对外提供服务的.
127.0.0.1:6380> set b world
(error) CLUSTERDOWN The cluster is down
```

### (12). 模拟Redis Master出现故障
> 结论:   
> 在Redis集群有Master和Slave,如果,下线Master,会触发Slave提升为Master,整个集群会有一小段时间无法提供服务,等到Slave提升为Master后,整个集群仍然可以提供服务.  

```
# 1. 查看redis集群slot分布情况
lixin-macbook:redis-cluster lixin$ ./bin/redis-cli -c -a 888888 -p 6380
127.0.0.1:6380> CLUSTER SLOTS
1) 1) (integer) 0
   2) (integer) 5460
   3) 1) "127.0.0.1"
      2) (integer) 6380
      3) "d4fb72a249a42ecfce9f5e6d90eab917ddf20c22"
   4) 1) "127.0.0.1"
      2) (integer) 6385
      3) "edaeeffe57750afe67957f9ebbbcb0d919e9baea"
2) 1) (integer) 10923
   2) (integer) 16383
   3) 1) "127.0.0.1"
      2) (integer) 6384
      3) "894425fde65f7da875d4d078f41e01381e52c6d9"
   4) 1) "127.0.0.1"
      2) (integer) 6382
      3) "8a9bf55091d15fd77de0bf7d4db05e09a5b682db"
3) 1) (integer) 5461
   2) (integer) 10922
   3) 1) "127.0.0.1"
      2) (integer) 6381
      3) "d36cf78e5481e488bc50dd31960044cc1e69522f"
   4) 1) "127.0.0.1"
      2) (integer) 6383
      3) "bf09356f37d20288d1a8cb5754b0f5b6afddfdae"


# 2. 查看添加的数据路由在哪台机器上(6384)
127.0.0.1:6380> get a
-> Redirected to slot [15495] located at 127.0.0.1:6384
"a"

# 3. kill掉6384端口对应的进程.
lixin-macbook:redis-cluster lixin$ ps -ef|grep redis|grep 6384
  501  1588     1   0 10:10下午 ??         0:01.10 /Users/lixin/Developer/redis-cluster/bin/redis-server *:6384 [cluster]
lixin-macbook:redis-cluster lixin$  kill $(ps -ef|grep redis|grep 6384|awk '{print $2}')

# 4. 检查集群状态,为fail
127.0.0.1:6380> CLUSTER INFO
cluster_state:fail

# 5. 尝试获取数据(提示集群已经down)
127.0.0.1:6380> get a
(error) CLUSTERDOWN The cluster is down

# 6. 稍等一段时间(30s),再次查看slot信息,发现:6384被剔除了,由6382进行管理
127.0.0.1:6380> CLUSTER SLOTS
1) 1) (integer) 0
   2) (integer) 5460
   3) 1) "127.0.0.1"
      2) (integer) 6380
      3) "d4fb72a249a42ecfce9f5e6d90eab917ddf20c22"
   4) 1) "127.0.0.1"
      2) (integer) 6385
      3) "edaeeffe57750afe67957f9ebbbcb0d919e9baea"
2) 1) (integer) 5461
   2) (integer) 10922
   3) 1) "127.0.0.1"
      2) (integer) 6381
      3) "d36cf78e5481e488bc50dd31960044cc1e69522f"
   4) 1) "127.0.0.1"
      2) (integer) 6383
      3) "bf09356f37d20288d1a8cb5754b0f5b6afddfdae"
3) 1) (integer) 10923
   2) (integer) 16383
   3) 1) "127.0.0.1"
      2) (integer) 6382
      3) "8a9bf55091d15fd77de0bf7d4db05e09a5b682db"

# 6. 集群恢复后,即可获取数据.   
127.0.0.1:6380> get a
-> Redirected to slot [15495] located at 127.0.0.1:6382
"a"
```

### (13). 上线一组Redis(Master和Slave)
> 总结:  
> 1. 添加节点为Master.  
> 2. 添加节点为Slave,并指定Master.  
> 3. reshard重新分配slot,Redis提供的工具,相比你自己要实现好一些.  

```
lixin-macbook:~ lixin$ cd ~/Developer/redis-cluster

# 1. 创建数据目录和日志目录
lixin-macbook:redis-cluster lixin$ mkdir -p  data/{6386,6387}
lixin-macbook:redis-cluster lixin$ mkdir -p  logs/{6386,6387}

# 2. 拷贝配置文件
lixin-macbook:redis-cluster lixin$ cp conf/redis-6385.conf ./conf/redis-6386.conf
lixin-macbook:redis-cluster lixin$ cp conf/redis-6386.conf ./conf/redis-6387.conf

# 3. 修改配置文件内容,请参考前面(此处略)

# 4. 启动所有的redis节点(6386/6387)(此处略)
lixin-macbook:redis-cluster lixin$ ps -ef|grep redis
  501  2264     1   0 10:34下午 ??         0:00.03 /Users/lixin/Developer/redis-cluster/bin/redis-server *:6380 [cluster]
  501  2266     1   0 10:34下午 ??         0:00.03 /Users/lixin/Developer/redis-cluster/bin/redis-server *:6381 [cluster]
  501  2268     1   0 10:34下午 ??         0:00.03 /Users/lixin/Developer/redis-cluster/bin/redis-server *:6382 [cluster]
  501  2270     1   0 10:34下午 ??         0:00.04 /Users/lixin/Developer/redis-cluster/bin/redis-server *:6383 [cluster]
  501  2272     1   0 10:34下午 ??         0:00.03 /Users/lixin/Developer/redis-cluster/bin/redis-server *:6384 [cluster]
  501  2274     1   0 10:34下午 ??         0:00.04 /Users/lixin/Developer/redis-cluster/bin/redis-server *:6385 [cluster]
  501  2276     1   0 10:34下午 ??         0:00.02 /Users/lixin/Developer/redis-cluster/bin/redis-server *:6386 [cluster]
  501  2278     1   0 10:34下午 ??         0:00.02 /Users/lixin/Developer/redis-cluster/bin/redis-server *:6387 [cluster]

# 5. 登录redis
lixin-macbook:redis-cluster lixin$ ./bin/redis-cli -c -a 888888 -p 6380

# 6. 查看slot信息.
127.0.0.1:6380> CLUSTER SLOTS
1) 1) (integer) 5461
   2) (integer) 10922
   3) 1) "127.0.0.1"
      2) (integer) 6381
      3) "d36cf78e5481e488bc50dd31960044cc1e69522f"
   4) 1) "127.0.0.1"
      2) (integer) 6383
      3) "bf09356f37d20288d1a8cb5754b0f5b6afddfdae"
2) 1) (integer) 0
   2) (integer) 5460
   3) 1) "127.0.0.1"
      2) (integer) 6380
      3) "d4fb72a249a42ecfce9f5e6d90eab917ddf20c22"
   4) 1) "127.0.0.1"
      2) (integer) 6385
      3) "edaeeffe57750afe67957f9ebbbcb0d919e9baea"
3) 1) (integer) 10923
   2) (integer) 16383
   3) 1) "127.0.0.1"
      2) (integer) 6382
      3) "8a9bf55091d15fd77de0bf7d4db05e09a5b682db"
   4) 1) "127.0.0.1"
      2) (integer) 6384
      3) "894425fde65f7da875d4d078f41e01381e52c6d9"

# 7. 添加Master节点到集群里
# 127.0.0.1:6386   : 新的节点
# 127.0.0.1:6380   : 集群里任意一个节点
lixin-macbook:redis-cluster lixin$ ./bin/redis-cli -c -a 888888 --cluster  add-node 127.0.0.1:6386 127.0.0.1:6380
# 添加节点到集群
>>> Adding node 127.0.0.1:6386 to cluster 127.0.0.1:6380
>>> Performing Cluster Check (using node 127.0.0.1:6380)
M: d4fb72a249a42ecfce9f5e6d90eab917ddf20c22 127.0.0.1:6380
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
M: d36cf78e5481e488bc50dd31960044cc1e69522f 127.0.0.1:6381
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: bf09356f37d20288d1a8cb5754b0f5b6afddfdae 127.0.0.1:6383
   slots: (0 slots) slave
   replicates d36cf78e5481e488bc50dd31960044cc1e69522f
S: edaeeffe57750afe67957f9ebbbcb0d919e9baea 127.0.0.1:6385
   slots: (0 slots) slave
   replicates d4fb72a249a42ecfce9f5e6d90eab917ddf20c22
S: 894425fde65f7da875d4d078f41e01381e52c6d9 127.0.0.1:6384
   slots: (0 slots) slave
   replicates 8a9bf55091d15fd77de0bf7d4db05e09a5b682db
M: 8a9bf55091d15fd77de0bf7d4db05e09a5b682db 127.0.0.1:6382
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
>>> Send CLUSTER MEET to node 127.0.0.1:6386 to make it join the cluster.
[OK] New node added correctly.

# 8. 查看集群节点信息
127.0.0.1:6380> CLUSTER NODES
c43cabb3036f42ed592b36eaa3282760ecdb3caa 127.0.0.1:6386@16386 master - 0 1626014633000 0 connected

# 9. 添加新节点slave(6387)与master(6386)进行绑定
#  127.0.0.1:6387   : 新的slave节点
#  127.0.0.1:6380   : 集群中任意一个节点即可
#  --cluster-slave  : 代表添加的节点是slave节点
# --cluster-master-id c43cabb3036f42ed592b36eaa3282760ecdb3caa    :  给新节点指定主节点id
ixin-macbook:redis-cluster lixin$ ./bin/redis-cli -c -a 888888 --cluster  add-node 127.0.0.1:6387 127.0.0.1:6380  --cluster-slave --cluster-master-id c43cabb3036f42ed592b36eaa3282760ecdb3caa
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
>>> Adding node 127.0.0.1:6387 to cluster 127.0.0.1:6380
>>> Performing Cluster Check (using node 127.0.0.1:6380)
M: d4fb72a249a42ecfce9f5e6d90eab917ddf20c22 127.0.0.1:6380
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
M: d36cf78e5481e488bc50dd31960044cc1e69522f 127.0.0.1:6381
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: bf09356f37d20288d1a8cb5754b0f5b6afddfdae 127.0.0.1:6383
   slots: (0 slots) slave
   replicates d36cf78e5481e488bc50dd31960044cc1e69522f
S: edaeeffe57750afe67957f9ebbbcb0d919e9baea 127.0.0.1:6385
   slots: (0 slots) slave
   replicates d4fb72a249a42ecfce9f5e6d90eab917ddf20c22
S: 894425fde65f7da875d4d078f41e01381e52c6d9 127.0.0.1:6384
   slots: (0 slots) slave
   replicates 8a9bf55091d15fd77de0bf7d4db05e09a5b682db
M: c43cabb3036f42ed592b36eaa3282760ecdb3caa 127.0.0.1:6386
   slots: (0 slots) master
M: 8a9bf55091d15fd77de0bf7d4db05e09a5b682db 127.0.0.1:6382
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
>>> Send CLUSTER MEET to node 127.0.0.1:6387 to make it join the cluster.
Waiting for the cluster to join
>>> Configure node as replica of 127.0.0.1:6386.
[OK] New node added correctly.


# 10. 注意(6386和6387已经是主从关系了.)
127.0.0.1:6380> CLUSTER NODES
73855ed05187e40ac4b84a59c2246ffa5a207b66 127.0.0.1:6387@16387 slave c43cabb3036f42ed592b36eaa3282760ecdb3caa 0 1626015421000 11 connected
c43cabb3036f42ed592b36eaa3282760ecdb3caa 127.0.0.1:6386@16386 master - 0 1626015419812 11 connected

# 11. 此时新添加的一组Redis还不能被使用,需要分配solt
# 11.1 查看现有slot情况
127.0.0.1:6380> CLUSTER SLOTS
1) 1) (integer) 5461
   2) (integer) 10922
   3) 1) "127.0.0.1"
      2) (integer) 6381
      3) "d36cf78e5481e488bc50dd31960044cc1e69522f"
   4) 1) "127.0.0.1"
      2) (integer) 6383
      3) "bf09356f37d20288d1a8cb5754b0f5b6afddfdae"
2) 1) (integer) 0
   2) (integer) 5460
   3) 1) "127.0.0.1"
      2) (integer) 6380
      3) "d4fb72a249a42ecfce9f5e6d90eab917ddf20c22"
   4) 1) "127.0.0.1"
      2) (integer) 6385
      3) "edaeeffe57750afe67957f9ebbbcb0d919e9baea"
3) 1) (integer) 10923
   2) (integer) 16383
   3) 1) "127.0.0.1"
      2) (integer) 6382
      3) "8a9bf55091d15fd77de0bf7d4db05e09a5b682db"
   4) 1) "127.0.0.1"
      2) (integer) 6384
      3) "894425fde65f7da875d4d078f41e01381e52c6d9"

# 11.2 查看所有的节点信息
127.0.0.1:6380> CLUSTER NODES
d36cf78e5481e488bc50dd31960044cc1e69522f 127.0.0.1:6381@16381 master - 0 1626016580000 2 connected 5461-10922
bf09356f37d20288d1a8cb5754b0f5b6afddfdae 127.0.0.1:6383@16383 slave d36cf78e5481e488bc50dd31960044cc1e69522f 0 1626016581000 4 connected
edaeeffe57750afe67957f9ebbbcb0d919e9baea 127.0.0.1:6385@16385 slave d4fb72a249a42ecfce9f5e6d90eab917ddf20c22 0 1626016581000 6 connected
73855ed05187e40ac4b84a59c2246ffa5a207b66 127.0.0.1:6387@16387 slave c43cabb3036f42ed592b36eaa3282760ecdb3caa 0 1626016580309 11 connected
894425fde65f7da875d4d078f41e01381e52c6d9 127.0.0.1:6384@16384 slave 8a9bf55091d15fd77de0bf7d4db05e09a5b682db 0 1626016581320 10 connected
c43cabb3036f42ed592b36eaa3282760ecdb3caa 127.0.0.1:6386@16386 master - 0 1626016582329 11 connected
d4fb72a249a42ecfce9f5e6d90eab917ddf20c22 127.0.0.1:6380@16380 myself,master - 0 1626016580000 1 connected 0-5460
8a9bf55091d15fd77de0bf7d4db05e09a5b682db 127.0.0.1:6382@16382 master - 0 1626016583342 10 connected 10923-16383

# 11.3 重新分配slot
# reshard             : slot迁移
# 127.0.0.1:6380      : 集群中随机一个节点即可
# --cluster-from      : 需要从哪些源节点上迁移slot,可从多个源节点完成迁移,以逗号隔开,传递的是节点的node id
# --cluster-to        : slot需要迁移的目的节点的node id,目的节点只能填写一个,不传递该参数的话,则会在迁移过程中提示用户输入.
# --cluster-slots     : 需要迁移的slot数量,不传递该参数的话,则会在迁移过程中提示用户输入.
# --cluster-yes       : 迁移前,还需要用户再次确认.
lixin-macbook:redis-cluster lixin$ ./bin/redis-cli -c -a 888888 --cluster reshard 127.0.0.1:6380 --cluster-from d36cf78e5481e488bc50dd31960044cc1e69522f,d4fb72a249a42ecfce9f5e6d90eab917ddf20c22,8a9bf55091d15fd77de0bf7d4db05e09a5b682db --cluster-to  c43cabb3036f42ed592b36eaa3282760ecdb3caa --cluster-slots 1024


# 12. 验证slot分配信息
# 从结果上能看得出来,redis集群是把某一片范围内的数据,分配在某个节点上.
# 
127.0.0.1:6380> CLUSTER SLOTS
1) 1) (integer) 6486
   2) (integer) 10922
   3) 1) "127.0.0.1"
      2) (integer) 6381
      3) "d36cf78e5481e488bc50dd31960044cc1e69522f"
   4) 1) "127.0.0.1"
      2) (integer) 6383
      3) "bf09356f37d20288d1a8cb5754b0f5b6afddfdae"

# **************************************************************************************
# 6386节点上,分配上更多的slot,感觉好像是从现有(3个)的节点上,各抽出:1024个slot到该节点.
2) 1) (integer) 0
   2) (integer) 1022
   3) 1) "127.0.0.1"
      2) (integer) 6386
      3) "c43cabb3036f42ed592b36eaa3282760ecdb3caa"
   4) 1) "127.0.0.1"
      2) (integer) 6387
      3) "73855ed05187e40ac4b84a59c2246ffa5a207b66"
3) 1) (integer) 5461
   2) (integer) 6485
   3) 1) "127.0.0.1"
      2) (integer) 6386
      3) "c43cabb3036f42ed592b36eaa3282760ecdb3caa"
   4) 1) "127.0.0.1"
      2) (integer) 6387
      3) "73855ed05187e40ac4b84a59c2246ffa5a207b66"
4) 1) (integer) 10923
   2) (integer) 11945
   3) 1) "127.0.0.1"
      2) (integer) 6386
      3) "c43cabb3036f42ed592b36eaa3282760ecdb3caa"
   4) 1) "127.0.0.1"
      2) (integer) 6387
      3) "73855ed05187e40ac4b84a59c2246ffa5a207b66"
# **************************************************************************************

5) 1) (integer) 1023
   2) (integer) 5460
   3) 1) "127.0.0.1"
      2) (integer) 6380
      3) "d4fb72a249a42ecfce9f5e6d90eab917ddf20c22"
   4) 1) "127.0.0.1"
      2) (integer) 6385
      3) "edaeeffe57750afe67957f9ebbbcb0d919e9baea"
6) 1) (integer) 11946
   2) (integer) 16383
   3) 1) "127.0.0.1"
      2) (integer) 6382
      3) "8a9bf55091d15fd77de0bf7d4db05e09a5b682db"
   4) 1) "127.0.0.1"
      2) (integer) 6384
      3) "894425fde65f7da875d4d078f41e01381e52c6d9"

# 13. 尝试插入数据到新的节点
127.0.0.1:6382> CLUSTER KEYSLOT bbb
(integer) 5287
127.0.0.1:6382> set bbb world
-> Redirected to slot [5287] located at 127.0.0.1:6380
OK
```
### (14). 下线一组服务
> 结论:  
> 在下线一组Redis时,要求Redis里的数据需要先作迁移,否则,下线失败.   

```
# 1.下线6386和6387的服务.
127.0.0.1:6380> CLUSTER NODES
d36cf78e5481e488bc50dd31960044cc1e69522f 127.0.0.1:6381@16381 master - 0 1626060322363 2 connected 6827-10922
73855ed05187e40ac4b84a59c2246ffa5a207b66 127.0.0.1:6387@16387 slave c43cabb3036f42ed592b36eaa3282760ecdb3caa 0 1626060003000 11 connected
c43cabb3036f42ed592b36eaa3282760ecdb3caa 127.0.0.1:6386@16386 master - 0 1626060004943 11 connected 0-1364 5461-6826 10923-12287

# 2. 下线slave节点
lixin-macbook:redis-cluster lixin$ ./bin/redis-cli -c -a 888888 --cluster del-node  127.0.0.1:6387 73855ed05187e40ac4b84a59c2246ffa5a207b66
>>> Removing node 73855ed05187e40ac4b84a59c2246ffa5a207b66 from cluster 127.0.0.1:6387
>>> Sending CLUSTER FORGET messages to the cluster...
>>> SHUTDOWN the node.

# 3. 下线master节点
lixin-macbook:redis-cluster lixin$ ./bin/redis-cli -c -a 888888 --cluster del-node  127.0.0.1:6386 c43cabb3036f42ed592b36eaa3282760ecdb3caa
>>> Removing node c43cabb3036f42ed592b36eaa3282760ecdb3caa from cluster 127.0.0.1:6386
# 下线失败,因为节点里有数据,请先做reshard
[ERR] Node 127.0.0.1:6386 is not empty! Reshard data away and try again.

# 4. 重新reshard
# 把master(c43cabb3036f42ed592b36eaa3282760ecdb3caa)节点(6386)上的slot,全部迁移到:
#   slave(d36cf78e5481e488bc50dd31960044cc1e69522f)节点(6381)上
./bin/redis-cli -c -a 888888 --cluster reshard 127.0.0.1:6380 --cluster-from c43cabb3036f42ed592b36eaa3282760ecdb3caa --cluster-to d36cf78e5481e488bc50dd31960044cc1e69522f  --cluster-slots 1365
./bin/redis-cli -c -a 888888 --cluster reshard 127.0.0.1:6380 --cluster-from c43cabb3036f42ed592b36eaa3282760ecdb3caa --cluster-to d36cf78e5481e488bc50dd31960044cc1e69522f  --cluster-slots 6826

# 5. 检查下6386是否有slot分配
127.0.0.1:6380> CLUSTER NODES
c43cabb3036f42ed592b36eaa3282760ecdb3caa 127.0.0.1:6386@16386 master - 0 1626061120200 11 connected

# 6. 重新下线master节点(移除成功)
lixin-macbook:redis-cluster lixin$ ./bin/redis-cli -c -a 888888 --cluster del-node  127.0.0.1:6386 c43cabb3036f42ed592b36eaa3282760ecdb3caa
>>> Removing node c43cabb3036f42ed592b36eaa3282760ecdb3caa from cluster 127.0.0.1:6386
>>> Sending CLUSTER FORGET messages to the cluster...
>>> SHUTDOWN the node.

# 7. 再次查看slot分配信息.
127.0.0.1:6380> CLUSTER SLOTS
1) 1) (integer) 0
   2) (integer) 1364
   3) 1) "127.0.0.1"
      2) (integer) 6381
      3) "d36cf78e5481e488bc50dd31960044cc1e69522f"
   4) 1) "127.0.0.1"
      2) (integer) 6383
      3) "bf09356f37d20288d1a8cb5754b0f5b6afddfdae"
2) 1) (integer) 5461
   2) (integer) 12287
   3) 1) "127.0.0.1"
      2) (integer) 6381
      3) "d36cf78e5481e488bc50dd31960044cc1e69522f"
   4) 1) "127.0.0.1"
      2) (integer) 6383
      3) "bf09356f37d20288d1a8cb5754b0f5b6afddfdae"
3) 1) (integer) 1365
   2) (integer) 5460
   3) 1) "127.0.0.1"
      2) (integer) 6380
      3) "d4fb72a249a42ecfce9f5e6d90eab917ddf20c22"
   4) 1) "127.0.0.1"
      2) (integer) 6385
      3) "edaeeffe57750afe67957f9ebbbcb0d919e9baea"
4) 1) (integer) 12288
   2) (integer) 16383
   3) 1) "127.0.0.1"
      2) (integer) 6382
      3) "8a9bf55091d15fd77de0bf7d4db05e09a5b682db"
   4) 1) "127.0.0.1"
      2) (integer) 6384
      3) "894425fde65f7da875d4d078f41e01381e52c6d9"
```
### (15). 重新平衡
```
# 1. 查看现有的solt
lixin-macbook:redis-cluster lixin$ ./bin/redis-cli -c -a 888888 -p 6380
127.0.0.1:6380> CLUSTER SLOTS
1) 1) (integer) 6486
   2) (integer) 10922
   3) 1) "127.0.0.1"
      2) (integer) 6381
      3) "d36cf78e5481e488bc50dd31960044cc1e69522f"
   4) 1) "127.0.0.1"
      2) (integer) 6383
      3) "bf09356f37d20288d1a8cb5754b0f5b6afddfdae"
2) 1) (integer) 1023
   2) (integer) 5460
   3) 1) "127.0.0.1"
      2) (integer) 6380
      3) "d4fb72a249a42ecfce9f5e6d90eab917ddf20c22"
   4) 1) "127.0.0.1"
      2) (integer) 6385
      3) "edaeeffe57750afe67957f9ebbbcb0d919e9baea"
3) 1) (integer) 11946
   2) (integer) 16383
   3) 1) "127.0.0.1"
      2) (integer) 6382
      3) "8a9bf55091d15fd77de0bf7d4db05e09a5b682db"
   4) 1) "127.0.0.1"
      2) (integer) 6384
      3) "894425fde65f7da875d4d078f41e01381e52c6d9"
4) 1) (integer) 0
   2) (integer) 1022
   3) 1) "127.0.0.1"
      2) (integer) 6386
      3) "c43cabb3036f42ed592b36eaa3282760ecdb3caa"
   4) 1) "127.0.0.1"
      2) (integer) 6387
      3) "73855ed05187e40ac4b84a59c2246ffa5a207b66"
5) 1) (integer) 5461
   2) (integer) 6485
   3) 1) "127.0.0.1"
      2) (integer) 6386
      3) "c43cabb3036f42ed592b36eaa3282760ecdb3caa"
   4) 1) "127.0.0.1"
      2) (integer) 6387
      3) "73855ed05187e40ac4b84a59c2246ffa5a207b66"
6) 1) (integer) 10923
   2) (integer) 11945
   3) 1) "127.0.0.1"
      2) (integer) 6386
      3) "c43cabb3036f42ed592b36eaa3282760ecdb3caa"
   4) 1) "127.0.0.1"
      2) (integer) 6387
      3) "73855ed05187e40ac4b84a59c2246ffa5a207b66"

# 2. 平衡集群中各个节点的slot数量
lixin-macbook:redis-cluster lixin$ ./bin/redis-cli -c -a 888888 --cluster rebalance 127.0.0.1:6380
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
>>> Performing Cluster Check (using node 127.0.0.1:6380)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
>>> Rebalancing across 4 nodes. Total weight = 4.00
Moving 342 slots from 127.0.0.1:6382 to 127.0.0.1:6386
######################################################################################################################################################################################################################################################################################################################################################
Moving 342 slots from 127.0.0.1:6380 to 127.0.0.1:6386
######################################################################################################################################################################################################################################################################################################################################################
Moving 341 slots from 127.0.0.1:6381 to 127.0.0.1:6386
#####################################################################################################################################################################################################################################################################################################################################################

# 3. 再次查看slot信息
127.0.0.1:6380> CLUSTER SLOTS
1) 1) (integer) 6827
   2) (integer) 10922
   3) 1) "127.0.0.1"
      2) (integer) 6381
      3) "d36cf78e5481e488bc50dd31960044cc1e69522f"
   4) 1) "127.0.0.1"
      2) (integer) 6383
      3) "bf09356f37d20288d1a8cb5754b0f5b6afddfdae"
2) 1) (integer) 1365
   2) (integer) 5460
   3) 1) "127.0.0.1"
      2) (integer) 6380
      3) "d4fb72a249a42ecfce9f5e6d90eab917ddf20c22"
   4) 1) "127.0.0.1"
      2) (integer) 6385
      3) "edaeeffe57750afe67957f9ebbbcb0d919e9baea"
3) 1) (integer) 12288
   2) (integer) 16383
   3) 1) "127.0.0.1"
      2) (integer) 6382
      3) "8a9bf55091d15fd77de0bf7d4db05e09a5b682db"
   4) 1) "127.0.0.1"
      2) (integer) 6384
      3) "894425fde65f7da875d4d078f41e01381e52c6d9"
4) 1) (integer) 0
   2) (integer) 1364
   3) 1) "127.0.0.1"
      2) (integer) 6386
      3) "c43cabb3036f42ed592b36eaa3282760ecdb3caa"
   4) 1) "127.0.0.1"
      2) (integer) 6387
      3) "73855ed05187e40ac4b84a59c2246ffa5a207b66"
5) 1) (integer) 5461
   2) (integer) 6826
   3) 1) "127.0.0.1"
      2) (integer) 6386
      3) "c43cabb3036f42ed592b36eaa3282760ecdb3caa"
   4) 1) "127.0.0.1"
      2) (integer) 6387
      3) "73855ed05187e40ac4b84a59c2246ffa5a207b66"
6) 1) (integer) 10923
   2) (integer) 12287
   3) 1) "127.0.0.1"
      2) (integer) 6386
      3) "c43cabb3036f42ed592b36eaa3282760ecdb3caa"
   4) 1) "127.0.0.1"
      2) (integer) 6387
      3) "73855ed05187e40ac4b84a59c2246ffa5a207b66"
```
### (16). 查看集群信息
```
lixin-macbook:redis-cluster lixin$ ./bin/redis-cli -c -a 888888 --cluster info 127.0.0.1:6380
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
127.0.0.1:6380 (d4fb72a2...) -> 1 keys | 4096 slots | 1 slaves.
127.0.0.1:6381 (d36cf78e...) -> 0 keys | 4096 slots | 1 slaves.
127.0.0.1:6382 (8a9bf550...) -> 1 keys | 4096 slots | 1 slaves.
127.0.0.1:6386 (c43cabb3...) -> 0 keys | 4096 slots | 1 slaves.
[OK] 2 keys in 4 masters.
0.00 keys per slot on average.
```
### (17). 检查集群
```
lixin-macbook:redis-cluster lixin$ ./bin/redis-cli -c -a 888888 --cluster check 127.0.0.1:6380
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
127.0.0.1:6380 (d4fb72a2...) -> 1 keys | 4096 slots | 1 slaves.
127.0.0.1:6381 (d36cf78e...) -> 0 keys | 4096 slots | 1 slaves.
127.0.0.1:6382 (8a9bf550...) -> 1 keys | 4096 slots | 1 slaves.
127.0.0.1:6386 (c43cabb3...) -> 0 keys | 4096 slots | 1 slaves.
[OK] 2 keys in 4 masters.
0.00 keys per slot on average.
>>> Performing Cluster Check (using node 127.0.0.1:6380)
M: d4fb72a249a42ecfce9f5e6d90eab917ddf20c22 127.0.0.1:6380
   slots:[1365-5460] (4096 slots) master
   1 additional replica(s)
M: d36cf78e5481e488bc50dd31960044cc1e69522f 127.0.0.1:6381
   slots:[6827-10922] (4096 slots) master
   1 additional replica(s)
S: bf09356f37d20288d1a8cb5754b0f5b6afddfdae 127.0.0.1:6383
   slots: (0 slots) slave
   replicates d36cf78e5481e488bc50dd31960044cc1e69522f
M: 8a9bf55091d15fd77de0bf7d4db05e09a5b682db 127.0.0.1:6382
   slots:[12288-16383] (4096 slots) master
   1 additional replica(s)
S: 894425fde65f7da875d4d078f41e01381e52c6d9 127.0.0.1:6384
   slots: (0 slots) slave
   replicates 8a9bf55091d15fd77de0bf7d4db05e09a5b682db
S: 73855ed05187e40ac4b84a59c2246ffa5a207b66 127.0.0.1:6387
   slots: (0 slots) slave
   replicates c43cabb3036f42ed592b36eaa3282760ecdb3caa
M: c43cabb3036f42ed592b36eaa3282760ecdb3caa 127.0.0.1:6386
   slots:[0-1364],[5461-6826],[10923-12287] (4096 slots) master
   1 additional replica(s)
S: edaeeffe57750afe67957f9ebbbcb0d919e9baea 127.0.0.1:6385
   slots: (0 slots) slave
   replicates d4fb72a249a42ecfce9f5e6d90eab917ddf20c22
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```
### (18). 总结
> Redis5.X已经不再需要Ruby,看来,它已经慢慢的把生态圈开始完善了.  