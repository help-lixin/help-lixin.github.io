---
layout: post
title: 'Docker 安装MySQL'
date: 2020-11-01
author: 李新
tags: Docker
---

### (1). 查看可用的MySQL版本
> https://hub.docker.com/_/mysql?tab=tags&page=1&ordering=name

```
lixin-macbook:~ lixin$ docker pull mysql:5.7.9
    5.7.9: Pulling from library/mysql
    Image docker.io/library/mysql:5.7.9 uses outdated schema1 manifest format. Please upgrade to a schema2 image for better future compatibility. More information at https://docs.docker.com/registry/spec/deprecated-schema-v1/
    d4bce7fd68df: Pull complete 
    a3ed95caeb02: Pull complete 
    01588229585e: Pull complete 
    ada32b818a1a: Pull complete 
    ac7528e308ac: Pull complete 
    44e3fb8779c7: Pull complete 
    bfcca86efc6a: Pull complete 
    32da415dff2e: Pull complete 
    aae6d9712a36: Pull complete 
    3148136ce9cc: Pull complete 
    Digest: sha256:7c583f1c8a1377c467a8ec54253eb322aabaed71b184536af76c7b38ba35117d
    Status: Downloaded newer image for mysql:5.7.9
    docker.io/library/mysql:5.7.9
```
### (2). 查看本地镜像
```
lixin-macbook:~ lixin$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
mysql               5.7.9               ec7e75e5260c        4 years ago         360MB
```
### (3). 创建容器
```
lixin-macbook:DockerWorkspace lixin$ pwd
    /Users/lixin/DockerWorkspace

## 创建目录    
lixin-macbook:DockerWorkspace lixin$ mkdir -p  mysql/{data,conf,logs}
    ## /Users/lixin/DockerWorkspace/mysql/conf
    ## /Users/lixin/DockerWorkspace/mysql/data
    ## /Users/lixin/DockerWorkspace/mysql/logs


## 创建配置文件
lixin-macbook:DockerWorkspace lixin$ touch mysql/conf/my.cnf

```
### (4).运行
```
lixin-macbook:DockerWorkspace lixin$ docker run -p 3307:3306 --name mysql -v /Users/lixin/DockerWorkspace/mysql/conf:/etc/mysql/conf.d -v /Users/lixin/DockerWorkspace/mysql/logs:/var/log/mysql -v /Users/lixin/DockerWorkspace/mysql/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=123456 -d mysql:5.7.9
```

```
-p     : 本地端口与容器端口映射
--name : 给容器定义名称
-e     : 设置参数
-v     : 将主机目录挂载到容器的目录
-d     : 后台运行容器
```
### (5). 启动/停止容器
```
docker start mysql
docker stop mysql
```
### (6).连接
```
lixin-macbook:data lixin$ mysql -u root -P3307 -h 127.0.0.1 -p
    Enter password: 
    Welcome to the MySQL monitor.  Commands end with ; or \g.
    Your MySQL connection id is 3
    Server version: 5.7.9 MySQL Community Server (GPL)

    Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.

    Oracle is a registered trademark of Oracle Corporation and/or its
    affiliates. Other names may be trademarks of their respective
    owners.

    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

    mysql> 
```