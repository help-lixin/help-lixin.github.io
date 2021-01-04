---
layout: post
title: 'Docker 自制JDK1.8镜像'
date: 2020-11-01
author: 李新
tags: Docker
---

### (1). 项目目录
> jdk-8u271-linux-x64.tar.gz是在Oracle官网下载的.

```
lixin-macbook:jdk lixin$ tree
.
├── Dockerfile
└── jdk-8u271-linux-x64.tar.gz
```
### (2). Dockerfile
```
# 基于镜像centos7
FROM centos:centos7

# 添加下载的JDK到opt目录
ADD jdk-8u271-linux-x64.tar.gz /opt/

# 添加环境变量
ENV JAVA_HOME=/opt/jdk1.8.0_271
ENV PATH=$PATH:$JAVA_HOME/bin:
ENV CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JAVA_HOME/jre/lib/rt.jar
```
### (3). 基于Dockerfile构建镜像
```
# 构建镜像
lixin-macbook:jdk lixin$ docker build . -t jdk:1.8
Sending build context to Docker daemon  143.1MB
Step 1/5 : FROM centos:centos7
 ---> 8652b9f0cb4c
Step 2/5 : ADD jdk-8u271-linux-x64.tar.gz /opt/
 ---> fe4c14f7b76f
Step 3/5 : ENV JAVA_HOME=/opt/jdk1.8.0_271
 ---> Running in 9a7b011ed286
Removing intermediate container 9a7b011ed286
 ---> becdbddc4a82
Step 4/5 : ENV PATH=$PATH:$JAVA_HOME/bin:
 ---> Running in 6c9fa3567754
Removing intermediate container 6c9fa3567754
 ---> e83e830ea546
Step 5/5 : ENV CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JAVA_HOME/jre/lib/rt.jar
 ---> Running in 7793438840cc
Removing intermediate container 7793438840cc
 ---> 6aac19cb29fb
Successfully built 6aac19cb29fb
Successfully tagged jdk:1.8
```
### (4). 创建容器并检查
```
# 创建交互式容器
lixin-macbook:jdk lixin$ docker run -it --rm --name java jdk:1.8 /bin/bash 

# 查看容器环境变量($JAVA_HOME)
[root@a24e2400824b /]# echo $JAVA_HOME
/opt/jdk1.8.0_271

# 查看容器环境变量($PATH)
[root@a24e2400824b /]# echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/opt/jdk1.8.0_271/bin

# 查看容器环境变量($CLASSPATH)
[root@a24e2400824b /]# echo $CLASSPATH
.:/opt/jdk1.8.0_271/lib/dt.jar:/opt/jdk1.8.0_271/lib/tools.jar:/opt/jdk1.8.0_271/jre/lib/rt.jar

# 查看JDK的版本
[root@a24e2400824b /]# java -version
java version "1.8.0_271"
Java(TM) SE Runtime Environment (build 1.8.0_271-b09)
Java HotSpot(TM) 64-Bit Server VM (build 25.271-b09, mixed mode)
```
