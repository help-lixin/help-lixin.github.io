---
layout: post
title: 'Zeebe源码编译与介绍(一)' 
date: 2022-02-02
author: 李新
tags:  Zeebe
---

### (1). 概述
在这一篇主要对Zeebe进行源码编译以及项目结构进行介绍.

### (2). 源码下载与编译
```
lixin-macbook:~ lixin$ cd ~/GitRepository/
lixin-macbook:GitRepository lixin$ git clone  https://github.com/help-lixin/zeebe.git
lixin-macbook:GitRepository lixin$ cd zeebe
lixin-macbook:zeebe lixin$ mvn clean install -DskipTests -X
```
### (3). Zeebe项目目录介绍
```
lixin-macbook:zeebe lixin$ tree -L 1
.
├── atomix
├── benchmarks
├── bom
├── bors.toml
├── bpmn-model                      # 解析xml为model
├── broker                          # brokder
├── build-tools
├── clients                         # java/go client
├── dispatcher
├── dist                            # 最终二进制文件
├── engine                          # 引擎
├── exporter-api                    # export api
├── exporters
├── expression-language
├── gateway                         # gateway
├── gateway-protocol                # gateway协议
├── gateway-protocol-impl           # gateway协议实现
├── img
├── journal
├── licenses
├── logstreams
├── monitor                         # 监听
├── msgpack-core                    # msgpack协议
├── msgpack-value
├── parent
├── pom.xml
├── protocol                        # 协议信息
├── protocol-asserts
├── protocol-impl
├── protocol-jackson
├── protocol-test-util
├── qa
├── samples                         # 案例
├── snapshot
├── test
├── test-util
├── transport                      # 传输层
├── util
└── zb-db
```
### (4). Zeebe二进制文件
```
lixin-macbook:zeebe lixin$ tree dist/target 
dist/target
├── camunda-cloud-zeebe-1.3.0-SNAPSHOT.jar
├── camunda-cloud-zeebe-1.3.0-SNAPSHOT.tar.gz
└── camunda-cloud-zeebe-1.3.0-SNAPSHOT.zip
```