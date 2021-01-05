---
layout: post
title: 'Docker Images 相关命令'
date: 2020-11-01
author: 李新
tags: Docker
---

### (1). 仓库注册账号(略)

### (2). 登录仓库
```
# 远程登录
lixin-macbook:~ lixin$ docker login docker.com
    Username: xxxxx
    Password: 
    Login Succeeded
```

### (3). 查看Docker工作目录
```
# docker工作环境目录(config.json/daemon.json)
lixin-macbook:~ lixin$ ll ~/.docker/
    drwxr-xr-x   6 lixin  staff   192 11  6 16:07 application-template/
    -rw-r--r--   1 lixin  staff   123 12 30 15:37 config.json
    drwxr-xr-x   3 lixin  staff    96 10 31 21:06 contexts/
    -rw-r--r--@  1 lixin  staff    36 11  5 23:28 daemon.json
    drwxr-xr-x   3 lixin  staff    96 12 30 15:34 run/
    drwxr-xr-x   3 lixin  staff    96 12 30 15:34 scan/

# docker socket
lixin-macbook:~ lixin$ ll /var/run/docker.sock 
```
### (4). 检索镜像
```
# 检索镜像
lixin-macbook:~ lixin$ docker search centos
```
### (5). 拉取镜像
```
# 拉取镜像
# lixin-macbook:~ lixin$ docker pull centos:centos7
# 拉取镜像(指定:注册地址名称/仓库名称/镜像名称:标签名称)
lixin-macbook:~ lixin$ docker pull docker.io/library/centos:centos7
    centos7: Pulling from library/centos
    Digest: sha256:0f4ec88e21daf75124b8a9e5ca03c37a5e937e0e108a255d890492430789b60e
    Status: Downloaded newer image for centos:centos7
    docker.io/library/centos:centos7
```
### (6). 检索本地镜像
```
# 查看本地所有镜像
# lixin-macbook:~ lixin$ docker images
# 查看指定的镜像
lixin-macbook:~ lixin$ docker images centos:centos7
    REPOSITORY   TAG       IMAGE ID       CREATED       SIZE
    centos       centos7   8652b9f0cb4c   6 weeks ago   204MB
```
### (7). 给本地镜像打标签 
> <font color='red'>注意:xxxx为你在:docker.io注册的账号名称.</font> 

```
# 给镜像(8652b9f0cb4c)打个标签.
lixin-macbook:~ lixin$ docker tag 8652b9f0cb4c docker.io/lixinhelp/centos7:centos7

# 查看所有的镜像信息
lixin-macbook:~ lixin$ docker images
    REPOSITORY               TAG              IMAGE ID       CREATED       SIZE
    lixinhelp/centos7       centos7          8652b9f0cb4c   6 weeks ago   204MB
    centos                   centos7          8652b9f0cb4c   6 weeks ago   204MB
```
### (8). 推送镜像到远程仓库
```
# 推送镜像到远程仓库:docker.io
lixin-macbook:~ lixin$ docker push lixinhelp/centos7:centos7
    The push refers to repository [docker.io/lixinhelp/centos7]
    174f56854903: Mounted from library/centos 
    centos7: digest: sha256:e4ca2ed0202e76be184e75fb26d14bf974193579039d5573fb2348664deef76e size: 529
```
### (9). 删除镜像
```
# 该命令,仅仅只是删除了镜像.
# lixin-macbook:~ lixin$ docker rmi  alpine:latest
lixin-macbook:~ lixin$ docker rmi  docker.io/library/alpine:latest
    Untagged: alpine:latest
    Untagged: alpine@sha256:3c7497bf0c7af93428242d6176e8f7905f2201d8fc5861f45be7a346b5f23436
    Deleted: sha256:389fef7118515c70fd6c0e0d50bb75669942ea722ccb976507d7b087e54d5a23
    Deleted: sha256:777b2c648970480f50f5b4d0af8f9a8ea798eea43dbcf40ce4a8c7118736bdcf


# 彻底删除镜像ID相关的镜像和文件.
lixin-macbook:~ lixin$ docker rmi -f 389fef711851
    Untagged: alpine:latest
    Untagged:   alpine@sha256:3c7497bf0c7af93428242d6176e8f7905f2201d8fc5861f45be7a346b5f23436
    Deleted: sha256:389fef7118515c70fd6c0e0d50bb75669942ea722ccb976507d7b087e54d5a23
    Deleted: sha256:777b2c648970480f50f5b4d0af8f9a8ea798eea43dbcf40ce4a8c7118736bdcf
```