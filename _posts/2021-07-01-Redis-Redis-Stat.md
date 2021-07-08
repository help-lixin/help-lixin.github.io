---
layout: post
title: 'Redis Stat 安装'
date: 2021-07-01
author: 李新
tags:  Redis
---

### (1). Redis Stat 是什么?
redis-stat是一个用Ruby语言开发的工具,主要用于对Redis进行监控.     
redis-stat本身基于Redis的INFO命令封装而成,而不会像其它工具那样,基于MONITOR命令的监控.    
我们只需要在一台机器上安装redis-stat,就可在这台机器上监控所有的redis实例当前时间的数据信息.     

### (2). Redis Stat 安装
```
lixin-macbook:~ lixin$ cd ~/GitRepository/
lixin-macbook:GitRepository lixin$ mkdir redis-stat
lixin-macbook:GitRepository lixin$ cd redis-stat/
lixin-macbook:redis-stat lixin$ wget https://github.com/junegunn/redis-stat/releases/download/0.4.14/redis-stat-0.4.14.jar
```
### (3). 运行redis-stat
```
# 运行(指定web端口:63790,间隔:1秒采集一次数据)
lixin-macbook:redis-stat lixin$ java -jar redis-stat-0.4.14.jar 127.0.0.1:6379 127.0.0.1:6380  --server=63790  1
Puma 2.3.2 starting...
* Min threads: 0, max threads: 16
* Environment: development
* Listening on tcp://0.0.0.0:63790
== Sinatra/1.3.6 has taken the stage on 63790 for production with backup from Puma
┌────────────────────────┬────────────────┬────────────────┐
│                        │ 127.0.0.1:6379 │ 127.0.0.1:6380 │
├────────────────────────┼────────────────┼────────────────┤
│          redis_version │          6.0.8 │          6.0.8 │
│             redis_mode │     standalone │     standalone │
│             process_id │           5197 │           6301 │
│      uptime_in_seconds │           2107 │             76 │
│         uptime_in_days │              0 │              0 │
│                   role │         master │         master │
│       connected_slaves │              0 │              0 │
│            aof_enabled │              0 │              0 │
│ rdb_bgsave_in_progress │              0 │              0 │
│     rdb_last_save_time │     1625299415 │     1625301446 │
└────────────────────────┴────────────────┴────────────────┘

┌────────┬──┬──┬──┬───┬──────┬──────┬────┬─────┬─────┬─────┬──────┬─────┬─────┬─────┐
     time us sy cl bcl    mem    rss keys cmd/s exp/s evt/s hit%/s hit/s mis/s aofcs
├────────┼──┼──┼──┼───┼──────┼──────┼────┼─────┼─────┼─────┼──────┼─────┼─────┼─────┤
 16:38:42  -  -  2   0 2.18MB 3.70MB    0     -     -     -      -     -     -    0B
 16:38:43  0  1  2   0 2.18MB 3.70MB    0  1.99     0     0      -     0     0    0B
```
### (4). 输出内容详解
```
time:  更新参数时间
us:    Redis服务器使用的用户CPU
sy:    Redis服务器使用的系统CPU
cl:    客户端连接数
bcl:   阻塞呼叫中挂起的客户端数量(BLPOP，BRPOP，BRPOPLPUSH)
mem:   Redis使用其分配程序分配的字节总数
rss:   从操作系统的角度,返回Redis已分配的内存总量.  
keys:   执行命令的key
cmd/s:   服务器已每秒执行的命令数量
exp/s:   因为过期而每秒被自动删除的数据库键数量
evt/s:   因为最大内存容量限制而每秒被驱逐(evict)的键数量
hit%/s:  查找数据库键成功的次数比例
hit/s:   查找数据库键成功的次数
mis/s:   查找数据库键每秒失败的次数
aofcs:   文件目前的大小
```
