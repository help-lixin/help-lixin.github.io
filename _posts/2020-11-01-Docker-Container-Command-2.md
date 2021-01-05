---
layout: post
title: 'Docker Container 相关命令(2)'
date: 2020-11-01
author: 李新
tags: Docker
---

### (1). 准备工作(下载Nginx镜像)
> ["Nginx"](https://hub.docker.com/_/nginx?tab=description&page=1&ordering=-last_updated)

```
# 下载nginx镜像
lixin-macbook:~ lixin$  docker pull nginx:1.19.6

# 查看镜像
lixin-macbook:~ lixin$ docker images |grep nginx
nginx    1.19.6              ae2feff98a0c   2 weeks ago    133MB

# 创建自己的标签
lixin-macbook:~ lixin$ docker tag ae2feff98a0c lixinhelp/nginx:1.19.6

lixin-macbook:~ lixin$ docker images |grep nginx
lixinhelp/nginx                                     1.19.6              ae2feff98a0c   2 weeks ago    133MB
nginx                                                1.19.6              ae2feff98a0c   2 weeks ago    133MB
lixin-macbook:~ lixin$ 
```
### (2). 宿主机端口与容器端口进行映射
> docker run -p 宿主机端口:容器内端口    

```
# 运行nginx镜像
# 宿主机8080与nginx容器80端口进行映射
lixin-macbook:~ lixin$ docker run --rm -d  --name nginx -p 8080:80 lixinhelp/nginx:1.19.6 
7637644be69f4b4bd4aca42192dca66fda7b2949c3827a47182e5049f4b5b0ff

# 查看镜像是否运行
lixin-macbook:~ lixin$ docker ps 
CONTAINER ID   IMAGE                     COMMAND                  CREATED         STATUS         PORTS                  NAMES
7637644be69f   lixinhelp/nginx:1.19.6   "/docker-entrypoint.…"   3 seconds ago   Up 2 seconds   0.0.0.0:8080->80/tcp   nginx


# 查看宿主机上所有开启的端口lixin-macbook:~ lixin$ netstat -AaLlnW
Current listen queue sizes (qlen/incqlen/maxqlen)
Socket           Flowhash Listen         Local Address         
f28c2482e4cb4419        0 0/0/128        *.8080                 
f28c2482ec478c09        0 0/0/10         127.0.0.1.4301         
f28c2482eca4fc09        0 0/0/128        127.0.0.1.4310         
f28c2482ed568e69        0 0/0/128        127.0.0.1.49578        
f28c2482e4cb2579        0 0/0/128        *.49152                                       
f28c2482e4cba229        0 0/0/128        *.49152                
f28c2482e4cbac09        0 0/0/128        *.22                   
f28c2482e4cb2b99        0 0/0/128        *.22              

# 测试宿主机8080端口
lixin-macbook:~ lixin$ curl -v  http://localhost:8080
*   Trying ::1...
* TCP_NODELAY set
* Connected to localhost (::1) port 8080 (#0)
> GET / HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.64.1
> Accept: */*
> 
< HTTP/1.1 200 OK
< Server: nginx/1.19.6
< Date: Mon, 04 Jan 2021 08:07:25 GMT
< Content-Type: text/html
< Content-Length: 612
< Last-Modified: Tue, 15 Dec 2020 13:59:38 GMT
< Connection: keep-alive
< ETag: "5fd8c14a-264"
< Accept-Ranges: bytes
```
### (3). 挂载容器内的目录与宿主机目录进行关联
> docker run -v 宿主机目录:容器内目录.  
>  准备工作(创建HTML目录和文件). 
 
```
# 宿主机工作环境
lixin-macbook:DockerWorkspace lixin$ pwd
/Users/lixin/DockerWorkspace

# 宿主机创建目录
lixin-macbook:DockerWorkspace lixin$ mkdir -p nginx/html

# 创建HTML
lixin-macbook:DockerWorkspace lixin$ vi nginx/html/index.html
<html>
   <head>
        <title>hello world</title>
   </head>
   <body>
      hello world
   </body>
</html>
```

> 挂载HTML到容器里,并启动.

```
# -v 宿主机目录:容器内目录
lixin-macbook:~ lixin$ docker run -d --rm  --name nginx  -p 8080:80  -v ~/DockerWorkspace/nginx/html:/usr/share/nginx/html lixinhelp/nginx:1.19.6 
a05b25cecd28b352739aef9ae43bff39f7dd9548883184f826323c033ad170e8

# 测试访问
lixin-macbook:DockerWorkspace lixin$ curl  http://localhost:8080
<html>
   <head>
        <title>hello world</title>
   </head>
   <body>
      hello world
   </body>
</html>
```
### (4). 传递环境变量到容器内部
> docker run -e 环境变量KEY=环境变量VALUE   

```
# -e 传递环境变量
# printenv : 打印环境变量
lixin-macbook:~ lixin$ docker run --rm -e MAIN_CLASS=help.lixin.Application -e JAVA_HOME=~/java/bin lixinhelp/centos7:centos7-net-tools printenv
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=678ee6ec3017
MAIN_CLASS=help.lixin.Application
JAVA_HOME=/Users/lixin/java/bin
HOME=/root
```
### (5). 