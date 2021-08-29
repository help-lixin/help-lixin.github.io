---
layout: post
title: 'Debezium源码编译(一)' 
date: 2021-08-29
author: 李新
tags:  Debezium
---

### (1). 编译要求
+ Git 2.2.1 or later
+ JDK 11 or later, e.g. OpenJDK
+ Apache Maven 3.6.3 or later
+ Docker Engine or Docker Desktop 1.9 or later

### (2). 源码编译
```
# 1. 进入下载目录
lixin-macbook:~ lixin$ cd ~/GitRepository/

# 2. 下载源码
lixin-macbook:debezium lixin$ git clone https://github.com/help-lixin/debezium.git

# 3. 进入源码目录
lixin-macbook:GitRepository lixin$ cd debezium/

# 4. 源码编译(-DskipITs:因为,本地没有Docke,跳过集成测试和docker来构建项目)
lixin-macbook:debezium lixin$ mvn clean install -DskipITs -DskipTests
```
