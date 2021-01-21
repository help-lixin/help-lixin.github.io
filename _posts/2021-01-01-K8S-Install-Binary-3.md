---
layout: post
title: 'Kubernetes 二进制安装之Docker安装与配置(三)'
date: 2021-01-01
author: 李新
tags: K8S
---

### (1). Docker安装
> 仅只需要在node-1和node-2节点安装docker

```
# 下载阿里云docker仓库
$ wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo

# 安装docker
$ yum -y install docker-ce-18.06.1.ce-3.el7

# 配置加速器(https://www.daocloud.io/mirror)
$ curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://f1361db2.m.daocloud.io

# 设置开机启动并运行docker
$ systemctl daemon-reload
$ systemctl enable docker 
$ systemctl start docker

$ docker -v
	Docker version 18.06.1-ce, build e68fc7a
```
