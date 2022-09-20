---
layout: post
title: 'Mac环境安装Protoc' 
date: 2022-09-19
author: 李新
tags:  GRPC
---


### (1). 概述
Protocol buffers是Google用于序列化结构化数据的语言中立、平台中立、可扩展的机制,比如XML，json，但更小、更快、更简单.只需定义一次数据的结构化方式,然后就可以使用生成的特殊源代码轻松地在各种数据流之间以及使用各种语言编写和读取结构化数据.  
### (2). 安装protoc
```
# 1. 搜索protobuf
lixin-macbook:Downloads lixin$ brew search protobuf
protobuf            protobuf-c          protobuf-swift      protobuf@3.6        protobuf@3.7        swift-protobuf

# 2. 安装最新protobuf
lixin-macbook:Downloads lixin$ brew install protobuf
```
### (3). 验证
```
lixin-macbook:~ lixin$ protoc --version
libprotoc 2.5.0
```