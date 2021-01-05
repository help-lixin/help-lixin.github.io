---
layout: post
title: 'Docker Container 相关命令(1)'
date: 2020-11-01
author: 李新
tags: Docker
---

### (1). 容器(Container)
> 容器(Container)是由Image派生而来的.

### (2). 查看运行的容器
```
# 查看所有运行中的容器
lixin-macbook:~ lixin$ docker ps 
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES

# 查看所有运行过的容器(不论是否存活)
lixin-macbook:~ lixin$ docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED       STATUS                     PORTS     NAMES
93cbed24ac0c   mysql:5.7.9   "/entrypoint.sh mysq…"   8 weeks ago   Exited (0) 6 weeks ago               slave
5a2704f5decf   mysql:5.7.9   "/entrypoint.sh mysq…"   8 weeks ago   Exited (137) 6 weeks ago             master
```

### (3). 启动容器(命令详解)
```
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]

OPTIONS选项:
-i       : 表示启动一个可交互的容器,并持续打开标准输入.
-t       : 表示使用本地终端关联到容器的标准输入输出上.
-d       : 表示将容器放置在后台运行.
--rm     : 退出后即删除之容器.
--name   : 表示定义容器唯一的名称.
IMAGE    : 表示要运行的镜像.
COMMAND  : 表示启动容器时要运行的命令.
```
### (4). 交互式启动一个容器
```
# 以终端的形式,运行一个容器(lixinhelp/centos7:centos7),容器的名称为:centos-7.
lixin-macbook:~ lixin$ docker run -it --name centos-7  lixinhelp/centos7:centos7  /bin/bash 

# 安装网络工具
[root@304b19fca4bc /]# yum -y install net-tools

# 查看IP地址
[root@304b19fca4bc /]# ifconfig                 
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.2  netmask 255.255.0.0  broadcast 172.17.255.255

# 查看Linux发行版
[root@304b19fca4bc /]# cat /etc/redhat-release 
CentOS Linux release 7.9.2009 (Core)
```
### (5). 非交互式方式启动容器
```
# 启动容器(lixinhelp/centos7:centos7),调用容器命令(echo hello),然后退出
# 退出时删除(--rm)容器.
lixin-macbook:~ lixin$ docker run --rm lixinhelp/centos7:centos7  /bin/echo test
test
```
### (6). 非交互式方式启动一个后台容器
```
# 以后台的方式启动容器(lixinhelp/centos7:centos7)
# 会返回一个容器ID
lixin-macbook:~ lixin$ docker run -d --name centos-7 lixinhelp/centos7:centos7 /bin/sleep 600000
e96cc40bbd940d372b24338c0560db9ca1b73b74ef97806bfdfcbef2eae3efc3

# 查看运行中的镜像
lixin-macbook:~ lixin$ docker ps 
CONTAINER ID   IMAGE                        COMMAND               CREATED         STATUS         PORTS     NAMES
e96cc40bbd94   lixinhelp/centos7:centos7   "/bin/sleep 600000"   4 seconds ago   Up 4 seconds             centos-7
```
### (7). 进入运行中的容器
```
# 查看运行中的容器(获得容器ID)
lixin-macbook:~ lixin$ docker ps
CONTAINER ID   IMAGE                        COMMAND               CREATED         STATUS         PORTS     NAMES
e96cc40bbd94   lixinhelp/centos7:centos7   "/bin/sleep 600000"   7 minutes ago   Up 6 minutes             centos-7


# 以交互式的方式进入容器(e96cc40bbd94为容器ID)
lixin-macbook:~ lixin$ docker exec -it e96cc40bbd94 /bin/bash 


# 查看容器IP地址
[root@e96cc40bbd94 /]# ifconfig 
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.2  netmask 255.255.0.0  broadcast 172.17.255.255
```
### (8). 停止运行中的容器
```
# 查看容器(获得容器ID)
lixin-macbook:~ lixin$ docker ps
CONTAINER ID   IMAGE                        COMMAND               CREATED          STATUS          PORTS     NAMES
e96cc40bbd94   lixinhelp/centos7:centos7   "/bin/sleep 600000"   10 minutes ago   Up 10 minutes             centos-7

# 停止指定容器ID的容器
lixin-macbook:~ lixin$ docker stop e96cc40bbd94
e96cc40bbd94

# 容器已经被停止了
lixin-macbook:~ lixin$ docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```
### (9). 重新启动曾经启动过的容器
```
# 重新启动指定容器ID的容器
# docker restart e96cc40bbd94
lixin-macbook:~ lixin$ docker start e96cc40bbd94
e96cc40bbd94

# 查看运行中的容器
lixin-macbook:~ lixin$ docker ps
CONTAINER ID   IMAGE                        COMMAND               CREATED          STATUS        PORTS     NAMES
e96cc40bbd94   lixinhelp/centos7:centos7   "/bin/sleep 600000"   12 minutes ago   Up 1 second             centos-7
```
### (10). 删除容器
```
# 停止容器
lixin-macbook:~ lixin$ docker stop e96cc40bbd94
e96cc40bbd94

# 删除容器(-f强制删除)
lixin-macbook:~ lixin$ docker rm  -f e96cc40bbd94
e96cc40bbd94

# 查看所有容器
lixin-macbook:~ lixin$ docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED       STATUS                     PORTS     NAMES
```
### (11). 提交容器内容到镜像
```
# 运行容器
lixin-macbook:~ lixin$ docker run -d --name centos-7 lixinhelp/centos7:centos7 /bin/sleep 60000000
ad5f439b7fbf297e175439bc1511d5433ceb7b4d5b237b6b10b05fd58e3cac39

# 进入容器
lixin-macbook:~ lixin$ docker exec -it ad5f439b7fbf /bin/bash 

# CentOS7默认是没有ifconfig命令,通过yum安装软件
[root@ad5f439b7fbf /]# yum -y install net-tools

# 一旦退出容器后,安装的软件在容器的可写层,并没有写入到镜像里.
# 所以,要在容器退出之前,提交容器信息到镜像里.
# 提交容器(ad5f439b7fbf),产生新的镜像(lixinhelp/centos7:centos7-net-tools)
lixin-macbook:~ lixin$ docker commit -p ad5f439b7fbf lixinhelp/centos7:centos7-net-tools
sha256:2b2f5db2242f478dd2d527f684841dca7afb41082ca9573933527874fa5e4985
 
# 提交新的镜像到远程仓库
lixin-macbook:~ lixin$ docker push lixinhelp/centos7:centos7-net-tools
The push refers to repository [docker.io/lixinhelp/centos7]
1e4025acf2d9: Pushing [======================>                            ]  42.81MB/94.28MB
174f56854903: Layer already exists 
1e4025acf2d9: Pushing [==========================================>        ]  80.06MB/94.28MB


# 创建容器(lixinhelp/centos7:centos7-net-tools),并进入容器.
lixin-macbook:~ lixin$ docker run -it lixinhelp/centos7:centos7-net-tools /bin/bash 
[root@781a212b604b /]# ifconfig 
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.2  netmask 255.255.0.0  broadcast 172.17.255.255
```
### (12). 导出镜像
```
# 查看有哪些镜像
lixin-macbook:~ lixin$ docker images
REPOSITORY                                           TAG                 IMAGE ID       CREATED          SIZE
lixinhelp/centos7                                   centos7-net-tools   2b2f5db2242f   41 minutes ago   298MB

# 导出镜像
lixin-macbook:~ lixin$ docker save 2b2f5db2242f > /tmp/centos-7-net-tools.tar

# 查看是否生成镜像成功
lixin-macbook:~ lixin$ ll /tmp/ |grep centos
-rw-r--r--  1 lixin  wheel  306056704  1  3 21:36 centos-7-net-tools.tar

```
### (13). 导入镜像
```
# 先删除镜像
lixin-macbook:~ lixin$ docker rmi -f lixinhelp/centos7:centos7-net-tools
	Untagged: lixinhelp/centos7:centos7-net-tools
	Untagged: lixinhelp/centos7@sha256:83f21db5717c334f45493c341b06a895c3b715d39246b595679000c00e44894c
	Deleted: sha256:2b2f5db2242f478dd2d527f684841dca7afb41082ca9573933527874fa5e4985

# 导入镜像
lixin-macbook:~ lixin$ docker load < /tmp/centos-7-net-tools.tar 
	Loaded image ID: sha256:2b2f5db2242f478dd2d527f684841dca7afb41082ca9573933527874fa5e4985

# 查看镜像(发现repository/tag都为空),需要设置一个tag
lixin-macbook:~ lixin$ docker images
REPOSITORY                                           TAG              IMAGE ID       CREATED        SIZE
<none>                                               <none>           2b2f5db2242f   18 hours ago   298MB


# 给指定的镜像ID设置tag
lixin-macbook:~ lixin$ docker tag 2b2f5db2242f lixinhelp/centos7:centos7-net-tools

# 再次查看镜像信息
lixin-macbook:~ lixin$ docker images
REPOSITORY                                           TAG                 IMAGE ID       CREATED        SIZE
lixinhelp/centos7                                   centos7-net-tools   2b2f5db2242f   18 hours ago   298MB
```
### (14). 查看容器日志
```
# 创建一个容器,输出:hello world,并将日志信息输出到/dev/null,而不在宿主机器上显示输出.
# 2 : 错误输出  
# 1 : 标准输出
# 把错误输出和标准输出合并,并写出到:/dev/null设备
lixin-macbook:~ lixin$ lixin-macbook:~ lixin$ docker run -d  --name centos-7 lixinhelp/centos7:centos7-net-tools /bin/echo "hello world" 2>&1 >> /dev/null

# 查看所有运行过的镜像
lixin-macbook:~ lixin$ docker ps -a
CONTAINER ID   IMAGE                                  COMMAND                  CREATED         STATUS                     PORTS     NAMES
f3e8e3e1dc74   lixinhelp/centos7:centos7-net-tools   "/bin/echo 'hello wo…"   5 seconds ago   Exited (0) 4 seconds ago             centos-7

# 查看某个镜像的日志信息(-f与tail -f相似)
lixin-macbook:~ lixin$ docker logs -f f3e8e3e1dc74
hello world
```
