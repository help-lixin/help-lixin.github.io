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

### (2). 修改源码

### (3). 源码编译
```
# 1. 进入下载目录
lixin-macbook:~ lixin$ cd ~/GitRepository/

# 2. 下载源码
lixin-macbook:debezium lixin$ git clone https://github.com/help-lixin/debezium.git

# 3. 进入源码目录
lixin-macbook:GitRepository lixin$ cd debezium/

# 4. 源码编译(-DskipITs:因为,本地没有Docke,跳过集成测试和docker来构建项目)
lixin-macbook:debezium lixin$ mvn clean install -DskipITs -DskipTests -Passembly 
```
### (4). 查看debezium源码目录
```
lixin-macbook:debezium lixin$ tree -L 1
.
├── debezium-api                                 # debezium-api
├── debezium-assembly-descriptors
├── debezium-bom
├── debezium-connector-mongodb                   # connector-mongodb
├── debezium-connector-mysql                     # connector-mysql
├── debezium-connector-oracle                    # connector-oracle
├── debezium-connector-postgres                  # connector-postgres
├── debezium-connector-sqlserver                 # connector-sqlserver
├── debezium-core                                # debezium核心代码
├── debezium-ddl-parser                          # debezium DDL解析器
├── debezium-e2e-benchmark
├── debezium-embedded
├── debezium-microbenchmark
├── debezium-microbenchmark-oracle
├── debezium-parent
├── debezium-quarkus-outbox
├── debezium-scripting
├── debezium-server
├── debezium-testing
├── documentation
├── github-support
├── jenkins-jobs
├── pom.xml
├── support
└── target
```
### (5). 查看kafka connector
```
# 查看debezium对kafka connector的扩展
lixin-macbook:debezium lixin$ ll debezium-connector-mysql/target/ |grep tar.gz
-rw-r--r--   1 lixin  staff  9096568  9 25 11:14 debezium-connector-mysql-1.7.0-SNAPSHOT-plugin.tar.gz
```