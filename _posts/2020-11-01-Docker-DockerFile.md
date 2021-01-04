---
layout: post
title: 'Docker DockerFile'
date: 2020-11-01
author: 李新
tags: Docker
---

### (1). Dockerfile是什么?
> Dockerfile是一个文本文件,里面实际就是:制作镜像的一堆指令集.  
> 可以根据Dockerfile文件,自动化构建出镜像(Image). 

### (2). Dockerfile的规则
> 1. # 为注释.  
> 2. 建议指令大写,内容小写.     
> 3. Docker是按顺序执行Dockerfile里的指令集的(从上到下依次执行).  
> 4. 每个Dockerfile的第一个指令(除了注释)必须是"FROM".用于为镜像文件构建过程中,指定的基准镜像.后续的指令运行于此基准镜像所提供的运行环境中.    

### (3). FROM指令
> 表示基础镜像来自哪里,本地镜像没有就从远程仓库获取,<font color='red'>指令必须放在最前面第一条.</font>  

```
# 代表基于centos7构建镜像
FROM centos:7
```

### (4). USER指令
> 用于指定进程使用什么用户运行,比如:nginx/mysql,类似于Linux下:useradd nginx. 

```
# 指定nginx启动使用nginx用户来运行
FROM hnlixin520/nginx:1.19.6
USER nginx
```
### (5). WORKDIR指令
> 用于指定切换到哪个目录.类似于Linux下:cd /usr/share/nginx/html        
```
# 指定基准镜像为:docker.com/hnlixin520/nginx:1.19.6
FROM docker.com/hnlixin520/nginx:1.19.6
# 启动进程的user为nginx
USER nginx
# 让容器切换到:/usr/share/nginx/html目录下,相当于:cd 
WORKDIR /usr/share/nginx/html
```

> 构建镜像(docker build . -t hnlixin520/nginx:1.19.6-2),注意,要当前目录下要有:Dockerfile文件.且这个文件名称的大小写都要一样的,不能更改.   

```
lixin-macbook:nginx lixin$ docker build . -t hnlixin520/nginx:1.19.6-2
Sending build context to Docker daemon  2.048kB
Step 1/3 : FROM hnlixin520/nginx:1.19.6
 ---> ae2feff98a0c
Step 2/3 : USER nginx
 ---> Running in 5681d8b61925
Removing intermediate container 5681d8b61925
 ---> 237999791749
Step 3/3 : WORKDIR /usr/share/nginx/html
 ---> Running in d58eb3549423
Removing intermediate container d58eb3549423
 ---> f16e8f49459d
Successfully built f16e8f49459d
Successfully tagged hnlixin520/nginx:1.19.6-2
```

> 查看镜像是否构建成功.

```
lixin-macbook:nginx lixin$ docker images|grep nginx
hnlixin520/nginx      1.19.6-2            f16e8f49459d   About a minute ago   133MB
hnlixin520/nginx      1.19.6              ae2feff98a0c   2 weeks ago          133MB

# 运行镜像
lixin-macbook:~ lixin$ docker run --rm --name nginx -it hnlixin520/nginx:1.19.6-2 /bin/bash 
# 查看当前用户名
nginx@a0854fcfe113:/usr/share/nginx/html$ whoami 
nginx

# 查看当前目录
nginx@a0854fcfe113:/usr/share/nginx/html$ pwd
/usr/share/nginx/html
```

### (6). ADD和COPY指令
> 复制文件到镜像里,区别是:ADD如果文件是可识别的压缩格式,则docker会帮忙解压缩的.

> 准备工作,创建index.html文件.

```
# 创建index.html
lixin-macbook:nginx lixin$ cat index.html 
<html>
   <head>
        <title>hello world</title>
   </head>
   <body>
      hello world
   </body>
</html>
```

> /Users/lixin/DockerWorkspace/DockerFiles/nginx/Dockerfile

```
# 指定基准镜像为:hnlixin520/nginx:1.19.6
FROM hnlixin520/nginx:1.19.6
# 启动进程的user为nginx
USER nginx
# 让容器切换到:/usr/share/nginx/html目录下
WORKDIR /usr/share/nginx/html
# 添加文件到:/usr/share/nginx/html目录下
ADD index.html /usr/share/nginx/html/
```

> 构建镜像,并打标签.

```
# 构建镜像并打标签
lixin-macbook:nginx lixin$ docker build . -t  hnlixin520/nginx:1.19.6-2
Sending build context to Docker daemon  3.072kB
Step 1/4 : FROM hnlixin520/nginx:1.19.6
 ---> ae2feff98a0c
Step 2/4 : USER nginx
 ---> Running in e58697e618ed
Removing intermediate container e58697e618ed
 ---> 27b1e01008fd
Step 3/4 : WORKDIR /usr/share/nginx/html
 ---> Running in 13405deaf3d5
Removing intermediate container 13405deaf3d5
 ---> 1164ba65cdc7
Step 4/4 : ADD index.html /usr/share/nginx/html/
 ---> dfdd108cd8b2
Successfully built dfdd108cd8b2
Successfully tagged hnlixin520/nginx:1.19.6-2


# 以交互式的方式运行容器,进入容器查看:index.html
lixin-macbook:nginx lixin$ docker run -it --rm --name nginx hnlixin520/nginx:1.19.6-2 /bin/bash 
nginx@ba2ffa28a2fd:/usr/share/nginx/html$ cat index.html 
<html>
   <head>
        <title>hello world</title>
   </head>
   <body>
      hello world
   </body>
</html>
```
### (7). EXPOSE指令
> 定义容器要暴露的端口,但是,只是内部暴露,外部映射还需要-p选项.

```
# 指定基准镜像为:hnlixin520/nginx:1.19.6
FROM hnlixin520/nginx:1.19.6
# 启动进程的user为nginx
USER nginx
# 让容器切换到:/usr/share/nginx/html目录下
WORKDIR /usr/share/nginx/html
# 添加文件到:/usr/share/nginx/html目录下
ADD index.html /usr/share/nginx/html/
# 暴露出容器的80端口
EXPOSE 80
```
### (8). RUN指令
>  执行相关的<font color='red'>系统命令.</font>   
>  <font color='red'>RUN指令的生命周期在构建镜像时触发的.</font>   

```
# 指定基准镜像为:centos:centos7
FROM centos:centos7
# 让容器运行系统命令(安装网络工具包)
RUN yum -y install net-tools
```

### (9). ENV指令
> 用于设置环境变量.   

```
# 指定基准镜像为:centos:centos7
FROM centos:centos7
# 让容器运行系统命令(安装网络工具包)
RUN yum -y install net-tools
# 设置环境变量
ENV JAVA_HOME /usr/java/bin
```

> 构建镜像

```
# 构建镜像
lixin-macbook:nginx lixin$ docker build . -t centos:centos7.1
Sending build context to Docker daemon  4.096kB
Step 1/3 : FROM centos:centos7
 ---> 8652b9f0cb4c
Step 2/3 : ENV JAVA_HOME /usr/java/bin
 ---> Running in 5145dd9bc811
Removing intermediate container 5145dd9bc811
 ---> 5fed737ec08b
Step 3/3 : RUN yum -y install net-tools
 ---> Running in 3c469058e619
Loaded plugins: fastestmirror, ovl
Determining fastest mirrors
 * base: mirrors.neusoft.edu.cn
 * extras: mirrors.163.com
 * updates: mirrors.163.com
Resolving Dependencies
--> Running transaction check
---> Package net-tools.x86_64 0:2.0-0.25.20131004git.el7 will be installed
--> Finished Dependency Resolution
Dependencies Resolved
================================================================================
 Package         Arch         Version                          Repository  Size
================================================================================
Installing:
 net-tools       x86_64       2.0-0.25.20131004git.el7         base       306 k

Transaction Summary
================================================================================
Install  1 Package

Total download size: 306 k
Installed size: 917 k
Downloading packages:
Public key for net-tools-2.0-0.25.20131004git.el7.x86_64.rpm is not installed
Retrieving key from file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
Importing GPG key 0xF4A80EB5:
 Userid     : "CentOS-7 Key (CentOS 7 Official Signing Key) <security@centos.org>"
 Fingerprint: 6341 ab27 53d7 8a78 a7c2 7bb1 24c6 a8a7 f4a8 0eb5
 Package    : centos-release-7-9.2009.0.el7.centos.x86_64 (@CentOS)
 From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : net-tools-2.0-0.25.20131004git.el7.x86_64                    1/1 
  Verifying  : net-tools-2.0-0.25.20131004git.el7.x86_64                    1/1 
Installed:
  net-tools.x86_64 0:2.0-0.25.20131004git.el7                                   
Complete!
Removing intermediate container 3c469058e619
 ---> af8b7fa94336
Successfully built af8b7fa94336
Successfully tagged centos:centos7.1

# 基于刚才的镜像创建容器,并进行容器内部
lixin-macbook:~ lixin$ docker run -it --rm --name centos centos:centos7.1 /bin/bash 

# 查看容器IP信息(ifconfig属于:net-tools)
[root@fc782965d88b /]# ifconfig 
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.2  netmask 255.255.0.0  broadcast 172.17.255.255

# 查看容器内部的环境变量
[root@fc782965d88b /]# printenv
HOSTNAME=fc782965d88b
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
# 这是我在ENV配置的环境变量.
JAVA_HOME=/usr/java/bin
_=/usr/bin/printenv
```
### (10). CMD指令
> 启动容器的时候要执行什么命令.CMD指令的生命周期在容器启动时.  
> <font color='red'>注意:当在交互模式(docker run -it)运行容器时,会覆盖CMD指令,而ENTRYPOINT指令则不会被覆盖.</font>  

```
# 指定基准镜像为:centos:centos7
FROM centos:centos7
# 让容器运行系统命令(安装网络工具包)
RUN yum -y install httpd
# 容器启动时执行的命令
CMD ["httpd","-D","FOREGROUND"]
```

> 构建镜像并测试

```
lixin-macbook:nginx lixin$ docker build . -t  httpd:1.0

Sending build context to Docker daemon   5.12kB
Step 1/3 : FROM centos:centos7
 ---> 8652b9f0cb4c
Step 2/3 : RUN yum -y install httpd
 ---> Running in 000663b6aa51
Loaded plugins: fastestmirror, ovl
Determining fastest mirrors
 * base: mirrors.aliyun.com
 * extras: mirrors.aliyun.com
 * updates: mirrors.aliyun.com
Resolving Dependencies
--> Running transaction check
---> Package httpd.x86_64 0:2.4.6-97.el7.centos will be installed
--> Processing Dependency: httpd-tools = 2.4.6-97.el7.centos for package: httpd-2.4.6-97.el7.centos.x86_64
--> Processing Dependency: system-logos >= 7.92.1-1 for package: httpd-2.4.6-97.el7.centos.x86_64
--> Processing Dependency: /etc/mime.types for package: httpd-2.4.6-97.el7.centos.x86_64
--> Processing Dependency: libaprutil-1.so.0()(64bit) for package: httpd-2.4.6-97.el7.centos.x86_64
--> Processing Dependency: libapr-1.so.0()(64bit) for package: httpd-2.4.6-97.el7.centos.x86_64
--> Running transaction check
---> Package apr.x86_64 0:1.4.8-7.el7 will be installed
---> Package apr-util.x86_64 0:1.5.2-6.el7 will be installed
---> Package centos-logos.noarch 0:70.0.6-3.el7.centos will be installed
---> Package httpd-tools.x86_64 0:2.4.6-97.el7.centos will be installed
---> Package mailcap.noarch 0:2.1.41-2.el7 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

================================================================================
 Package            Arch         Version                    Repository     Size
================================================================================
Installing:
 httpd              x86_64       2.4.6-97.el7.centos        updates       2.7 M
Installing for dependencies:
 apr                x86_64       1.4.8-7.el7                base          104 k
 apr-util           x86_64       1.5.2-6.el7                base           92 k
 centos-logos       noarch       70.0.6-3.el7.centos        base           21 M
 httpd-tools        x86_64       2.4.6-97.el7.centos        updates        93 k
 mailcap            noarch       2.1.41-2.el7               base           31 k

Transaction Summary
================================================================================
Install  1 Package (+5 Dependent packages)

Total download size: 24 M
Installed size: 32 M
Downloading packages:
Public key for httpd-tools-2.4.6-97.el7.centos.x86_64.rpm is not installed
Public key for apr-util-1.5.2-6.el7.x86_64.rpm is not installed
--------------------------------------------------------------------------------
Total                                              616 kB/s |  24 MB  00:40     
Retrieving key from file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
Importing GPG key 0xF4A80EB5:
 Userid     : "CentOS-7 Key (CentOS 7 Official Signing Key) <security@centos.org>"
 Fingerprint: 6341 ab27 53d7 8a78 a7c2 7bb1 24c6 a8a7 f4a8 0eb5
 Package    : centos-release-7-9.2009.0.el7.centos.x86_64 (@CentOS)
 From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : apr-1.4.8-7.el7.x86_64                                       1/6 
  Installing : apr-util-1.5.2-6.el7.x86_64                                  2/6 
  Installing : httpd-tools-2.4.6-97.el7.centos.x86_64                       3/6 
  Installing : centos-logos-70.0.6-3.el7.centos.noarch                      4/6 
  Installing : mailcap-2.1.41-2.el7.noarch                                  5/6 
  Installing : httpd-2.4.6-97.el7.centos.x86_64                             6/6 
  Verifying  : mailcap-2.1.41-2.el7.noarch                                  1/6 
  Verifying  : apr-1.4.8-7.el7.x86_64                                       2/6 
  Verifying  : apr-util-1.5.2-6.el7.x86_64                                  3/6 
  Verifying  : httpd-2.4.6-97.el7.centos.x86_64                             4/6 
  Verifying  : httpd-tools-2.4.6-97.el7.centos.x86_64                       5/6 
  Verifying  : centos-logos-70.0.6-3.el7.centos.noarch                      6/6 

Installed:
  httpd.x86_64 0:2.4.6-97.el7.centos                                            

Dependency Installed:
  apr.x86_64 0:1.4.8-7.el7                                                      
  apr-util.x86_64 0:1.5.2-6.el7                                                 
  centos-logos.noarch 0:70.0.6-3.el7.centos                                     
  httpd-tools.x86_64 0:2.4.6-97.el7.centos                                      
  mailcap.noarch 0:2.1.41-2.el7                                                 

Complete!
Removing intermediate container 000663b6aa51
 ---> abf4d2e51be4
Step 3/3 : CMD ["httpd","-D","FOREGROUND"]
 ---> Running in b814cc7f4e59
Removing intermediate container b814cc7f4e59
 ---> a65dc0831ecb
Successfully built a65dc0831ecb
Successfully tagged httpd:1.0

# 创建容器(指定宿主机与容器的80端口关联)
lixin-macbook:nginx lixin$ docker run -p:8080:80  -d --rm --name httpd httpd:1.0
480c381d61fc5de905f30eb1b8b58755e907dc9c138621371d3badbf5122446a

# 查看容器是否运行
lixin-macbook:nginx lixin$ docker ps 
CONTAINER ID   IMAGE       COMMAND                 CREATED          STATUS          PORTS                  NAMES
480c381d61fc   httpd:1.0   "httpd -D FOREGROUND"   26 seconds ago   Up 25 seconds   0.0.0.0:8080->80/tcp   httpd

# 访问8080端口
lixin-macbook:nginx lixin$ curl -v http://localhost:8080
*   Trying ::1...
* TCP_NODELAY set
* Connected to localhost (::1) port 8080 (#0)
> GET / HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.64.1
> Accept: */*
```
### (11). ENTRYPOINT指令
> ENTRYPOINT与CMD一样,也是最后一条命令执行,但是和CMD不同的是,CMD会被docker run中执行的命令给覆盖.

> Dockerfile

```
# 指定基准镜像为:centos:centos7
FROM centos:centos7

# 添加sh到容器里(注意:sh要有可执行权限)
ADD entrypoint.sh /entrypoint.sh

# 让容器运行系统命令(安装网络工具包)
RUN yum -q -y install epel-release && yum -y install nginx
RUN chmod 755 /entrypoint.sh

# 容器启动时执行的命令
ENTRYPOINT /entrypoint.sh
```

>  entrypoint.sh 

```
#!/bin/bash
/sbin/nginx -g "daemon off;"
```
> 构建镜像并测试

```
# 构建镜象
lixin-macbook:nginx lixin$ docker build . -t nginx:laster 

# 以后台的方式运行容器,并与宿主机8080端口绑定
lixin-macbook:nginx lixin$ docker run -d  -p 8080:80 --rm --name nginx nginx:laster 
040067f207533973bb702cbb458ae4e6cea0830e13a74cc13c118db2a905c053

# 查看docker运行容器
lixin-macbook:nginx lixin$ docker ps -a
CONTAINER ID   IMAGE          COMMAND                  CREATED              STATUS                     PORTS                  NAMES
040067f20753   nginx:laster   "/bin/sh -c /entrypo…"   About a minute ago   Up About a minute          0.0.0.0:8080->80/tcp   nginx

# 测试访问8080端口
lixin-macbook:nginx lixin$ curl http://localhost:8080
<html>
<head><title>403 Forbidden</title></head>
<body>
<center><h1>403 Forbidden</h1></center>
<hr><center>nginx/1.16.1</center>
</body>
</html>
```
