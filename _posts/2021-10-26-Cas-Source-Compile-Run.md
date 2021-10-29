---
layout: post
title: 'CAS源码编译并运行(二)' 
date: 2021-10-26
author: 李新
tags:  CAS
---

### (1). 概述
在这里我使用cas-overlay-template来搭建Cas.  

### (2). 环境要求
+ JDK11

### (3). 源码构建
```
# 1. 进入工作目录
lixin@lixin ~ % cd ~/GitWorkspace
# 2. 下载源码
lixin@lixin GitWorkspace % git clone https://github.com/help-lixin/cas-overlay-template.git
# 3. 进入cas-overlay-template目录
lixin@lixin GitWorkspace % cd cas-overlay-template
# 4. 构建源码(构建过程会比较漫长)
lixin@lixin cas-overlay-template % ./gradlew clean build -x test
```
### (4). 配置登录的账号和密码
```
infinova@lixin cas-overlay-template % more  src/main/resources/application.yml
# Application properties that need to be
# embedded within the web application can be included here
cas:
  authn:
    accept:
      users: lixin::123456
```
### (5). 创建配置
```
# 拷贝源码目录下的配置到指定配置目录
infinova@lixin cas-overlay-template % sudo cp -rf etc/cas /etc
# 修改目录权限
lixin@lixin cas-overlay-template % sudo chown -R lixin:staff /etc/cas
```
### (6). 自签证书
```
#  自签证书
infinova@lixin cas-overlay-template % ./gradlew createKeystore
```
### (7). 运行构建后的应用
```
lixin@lixin cas-overlay-template % java -Xdebug -Xrunjdwp:transport=dt_socket,address=5000,server=y,suspend=y -jar ./build/libs/cas.war
```
### (8). WEB测试
!["cas 登录"](/assets/cas/imgs/cas-login.png)  
!["cas 首页"](/assets/cas/imgs/cas-home.png)   