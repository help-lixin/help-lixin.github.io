---
layout: post
title: 'RabbitMQ Docker单机安装(二)' 
date: 2021-10-01
author: 李新
tags:  RabbitMQ
---

### (1). 概述
由于自己的机器是Mac,安装软件比较麻烦,而通过Docker安装软件后卸载软件也比较简单,所以,在搭建集群之前,都用单机版做学习,后面会专有和篇来安装集群版.   

### (2). 镜像拉取
> 选择一个带着management(UI)功能的,[参考地址](https://www.rabbitmq.com/download.html)  

```
lixin@lixin ~ % docker pull rabbitmq:3.9.7-management
```
### (3). 启动容器
```
# 1. 创建容器
lixin@lixin ~ %  docker run -d --name rabbitmq3.9.7 -p 5672:5672 -p 15672:15672 --hostname rabbitmq -e RABBITMQ_DEFAULT_VHOST=test_vhost  -e RABBITMQ_DEFAULT_USER=admin -e RABBITMQ_DEFAULT_PASS=admin rabbitmq:3.9.7-management

# 2. 检查运行情况
lixin@lixin ~ % docker ps
CONTAINER ID   IMAGE                       COMMAND                  CREATED          STATUS         PORTS                                                                                                                                                 NAMES
476d87998331   rabbitmq:3.9.7-management   "docker-entrypoint.s…"   10 seconds ago   Up 9 seconds   4369/tcp, 5671/tcp, 0.0.0.0:5672->5672/tcp, :::5672->5672/tcp, 15671/tcp, 15691-15692/tcp, 25672/tcp, 0.0.0.0:15672->15672/tcp, :::15672->15672/tcp   rabbitmq3.9.7
```
### (4). 访问WEB UI
!["RabbitMQ登录页面"](/assets/rabbitmq/imgs/rabbitmq-login.png)   
!["RabbitMQ首页"](/assets/rabbitmq/imgs/rabbitmq-home.png)   