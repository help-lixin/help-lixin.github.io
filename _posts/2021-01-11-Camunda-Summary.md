---
layout: post
title: 'Camunda 源码下载与编译(一)'
date: 2021-01-11
author: 李新
tags: Camunda
---

### (1). 为什么要用Camunda?
> 在前面我有分析过:Activiti的问题.在我的角度来看,Activiti没有跟上微服务的脚步,而Camunda都解决了那些问题.

### (2). Camunda源码下载
```
$ git clone -b 7.14.0    https://github.com/camunda/camunda-bpm-platform.git   camunda-bpm-platform-7.14.0
$ cd camunda-bpm-platform-7.14.0
$ mvn clean install -DskipTests -Pdistro
```
### (3). 查看分发文件
```
# 查看最后生成二进是路径地址
lixin-macbook:camunda-bpm-platform-7.14.0 lixin$ ll ./distro/tomcat/distro/target/
-rw-r--r--  1 lixin  staff  85642687 Jan  5 16:07 camunda-bpm-tomcat-7.14.0.tar.gz
-rw-r--r--  1 lixin  staff  86668608 Jan  5 16:07 camunda-bpm-tomcat-7.14.0.zip
```

### (4). camunda-bpm-tomcat-7.14.0 目录
```
lixin-macbook:Desktop lixin$ tree camunda-bpm-tomcat-7.14.0 -L 1
camunda-bpm-tomcat-7.14.0
├── camunda-h2-dbs                 # h2数据存储
├── lib
├── server                         # 内部实际是一个tomcat
├── shutdown-camunda.bat
├── shutdown-camunda.sh
├── sql
├── start-camunda.bat
└── start-camunda.sh
```

### (4). 访问
> 账号和密码都是(demo/demo)
["http://localhost:8080/camunda-welcome/index.html"](http://localhost:8080/camunda-welcome/index.html)


!["camunda-welcome"](/assets/camunda/imgs/camunda-welcome.jpg)

!["camunda-admin"](/assets/camunda/imgs/camunda-admin.jpg)

!["camunda-tasklist"](/assets/camunda/imgs/camunda-tasklist.png)
