---
layout: post
title: 'X-Pipe 介绍(一)' 
date: 2021-08-21
author: 李新
tags:  X-Pipe 
---

### (1). X-Pipe 是什么
X-Pipe是由携程框架部门研发的Redis多数据中心复制管理系统.基于Redis的Master-Slave复制协议,实现低延时、高可用的Redis多数据中心、跨公网数据复制,并且提供一键机房切换,复制监控、异常报警等功能.开源版本和携程内部生产环境版本一致. 

### (2). X-Pipe架构图
+ Console 
  - 用来管理多机房的元信息数据,同时提供用户界面,供用户进行配置和DR切换等操作.
+ Keeper
  - 负责缓存Redis操作日志,并对跨机房传输进行压缩、加密等处理. 
+ Meta Server
  - 管理单机房内的所有keeper状态,并对异常状态进行纠正. 

!["X-Pipe Architecture"](/assets/x-pipe/imgs/x-pipe-architecture.jpeg)  

### (3). Redis数据复制问题
多数据中心首先要解决的是数据复制问题,即数据如何从一个DC传输到另外一个DC.X-Pipe决定采用伪slave的方案,即实现Redis协议,伪装成为Redis Slave,让Redis Master推送数据至伪Slave,这个伪slave,X-Pipe把它称为keeper,如下图所示:  

!["Keeper"](/assets/x-pipe/imgs/x-pipe-keeper.jpeg)   

Keeper带来的优势:
+ 减少与Master全量同步
  - 如果异地机房Slave直接连向Master,多个Slave会导致Master多次全量同步,而Keeper可以缓存rdb和replication log,异地机房的Slave直接从Keeper获取数据,增强Master的稳定性.  
+ 减少多数据中心网络流量
  - 在两个数据中心之间,数据只需要通过Keeper传输一次,且Keeper之间传输协议可以自定义,方便支持压缩.  
+ 网络异常时减少全量同步
  - Keeper将Redis日志数据缓存到磁盘,这样,可以缓存大量的日志数据(Redis将数据缓存到内存ring buffer,容量有限),当数据中心之间的网络出现较长时间异常时仍然可以续传日志数据.  
+ 安全性提升
  - 多个机房之间的数据传输往往需要通过公网进行,这样数据的安全性变得极为重要,keeper之间的数据传输也可以加密(暂未实现),提升安全性.  
### (4). 源码下载与编译
```
# 1. jdk
# 2. maven
# 3. 清空本地 maven 仓库的 com 和 org 目录

# 4. clone x-pipe的分支:mvn_repo
[root@erp-100 ~]# git clone -b mvn_repo git@github.com:help-lixin/x-pipe.git x-pipe-mvn-repo
[root@erp-100 ~]# cd x-pipe-mvn-repo/
[root@erp-100 x-pipe-mvn-repo]# sh install.sh

# 5. x-pip依赖cat,所以,在打包前,先本地安装cat
[root@erp-100 ~]# git clone git@github.com:help-lixin/cat.git
[root@erp-100 ~]# cd cat/


# 6. clone x-pipe
# 编译了一个下午,还是没有编译通过,想放弃的心都有了.
#  Could not resolve dependencies for project com.ctrip.framework.xpipe:xpipe-parent:pom:1.2.4: com.dianping.cat:cat-client:jar:3.0.0
[root@erp-100 ~]# git clone git@github.com:help-lixin/x-pipe.git
[root@erp-100 ~]# cd x-pipe
[root@erp-100 x-pipe]# mvn clean install -Plocal,package -DskipTests
```
### (5). 项目结构如下
```
lixin-macbook:x-pipe lixin$ tree -L 2
.
├── core                                                    # 最核心的底层依赖
│   ├── pom.xml
│   ├── src
│   └── target
├── redis                                                   # redis交互的组件
│   ├── redis-checker
│   ├── redis-console                                       # console主要负责整个系统元信息的管理,比如cluster、shard、redis、keeper.并且提供系统监控、报警等功能.
│   ├── redis-core                                          # redis相关核心依赖
│   ├── redis-integration-test                              # 集成测试相关用例
│   ├── redis-keeper                                        # keeper主要实现redis数据复制协议,向redis master请求数据,并且将数据传播至slave,用于缓存redis复制日志以及rdb数据.  
│   ├── redis-meta                                          # meta server主要有两部分功能:一部分是和console交互,当console配置信息变化时,meta server调用keeper container以及keeper相关的接口.执行这部分变化.第二部分是管理单机房内的所有 keeper、Redis的 状态,并对异常状态进行纠正.  
│   ├── redis-proxy
│   └── redis-proxy-client
├── services                                                # 没太理解xpipe对这部份功能的抽象
│   ├── ctrip-integration-test
│   ├── ctrip-service
│   ├── local-service
│   └── pom.xml
└── xpipe-stype.xml
```
