---
layout: post
title: 'RedisLive 安装'
date: 2021-07-01
author: 李新
tags:  Redis
---

### (1). RedisLive是什么? 
> RedisLive提供对Redis实例的内存使用情况,接收的客户端命令,接收的请求数量以及键进行监控. 
> RedisLive的工作原理基于Redis的INFO和MONITOR命令,通过向Redis实例发送INFO和MONITOR命令来获取Redis实例当前的运行数据.  
### (2). RedisLive安装
```
lixin-macbook:~ lixin$ cd ~/GitRepository/
# 1. 下载源码
lixin-macbook:GitRepository lixin$ git clone git@github.com:snakeliwei/RedisLive.git
lixin-macbook:GitRepository lixin$ cd RedisLive/
# 2. 安装依赖
lixin-macbook:RedisLive lixin$ pip install -r requirements.txt -i http://pypi.douban.com/simple/ --trusted-host pypi.douban.com
# 3. 创建配置文件
lixin-macbook:RedisLive lixin$ cp src/redis-live.conf.example src/redis-live.conf
lixin-macbook:RedisLive lixin$ cd src/
```
### (3). RedisLive配置(redis-live.conf)
```
lixin-macbook:src lixin$ cat redis-live.conf
{
	"RedisServers":
	[
		{
  			"server": "127.0.0.1",
  			"port" : 6379
		}
	],
	"DataStoreType" : "sqlite",
	"SqliteStatsStore" :
	{
		"path":  "db/redislive.sqlite"
	}
}
```
### (4). 启动redis-monitor
```
# redis-monitor.py用于向Redis实例发送INFO和MONITOR命令并获取其返回结果.  
# duration参数指定了监控脚本的运行持续时间,例如设置为120秒,即经过120秒后,监控脚本会自动退出,并在终端打印shutting down… 的提示.
lixin-macbook:src lixin$ ./redis-monitor.py --duration=120
```
### (5). 启动redis-live
```
lixin-macbook:src lixin$ ./redis-live.py
```
### (6). 访问redis-live
```
http://localhost:8888/index.html
```