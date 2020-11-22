---
layout: post
title: 'Jmeter插件安装'
date: 2018-03-24
author: 李新
tags: Jmeter
---

### (1). 下载插件管理包并安装
> https://jmeter-plugins.org/install/Install/
> 拷贝jmeter-plugins-manager-1.4.jar 到 ${JMETER_HOME}/lib/ext目录里
> 重启jemeter


### (2). Jmeter Plugins Manager
> 1. 启动Jmeter -> 选项 -> Plugins Manager

### (3). 安装插件(Basic Grapsh)
> Basic Grapsh插件包含如下三个组件:( Transactions per Second [每秒事务数]/ Response Times Over Time [事务响应时间]/ Active Threads Over Time[每秒活动线程数] ) 

### (4). 安装插件(PerfMon Metrics Collector)
> 1. PerfMon Metrics Collector:即服务器性能监控数据采集,所以,需要在服务器端安装agent插件
> 2. 安装插件PerfMon Metrics Collector   
> 3. 下载ServerAgent(https://github.com/undera/perfmon-agent)    
> 4. 上传ServerAgent到压测服务器   
> 5. 启动ServerAgent(./startAgent.sh --udp-port 0 --tcp-port 3456)
> 6. 启用监听器 -> PerfMon Metrics Collector -> 配置:Server to Monitor(127.0.0.1:3456)

### (5). 
### (6). 
### (7). 
### (8). 
### (9). 
### (10). 