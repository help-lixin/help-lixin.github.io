---
layout: post
title: 'Seata 源码下载并编译(五)'
date: 2021-01-28
author: 李新
tags: Seata
---

### (1). 下载源码
```
# seata源码
$ git clone -b v1.4.0 https://github.com/seata/seata.git seata-v1.4.0

# seata案例源码
$ git clone https://github.com/seata/seata-samples.git
```
### (2). 编译源码
```
$ cd seata-v1.4.0
# 编译源码
$ mvn clean install -DskipTests -Prelease-seata
```
### (3). 目录介绍
> Seata好像没有给出像dubbo那样详细的文档来说明模块之间的依赖关系,只能自己画个依赖图出来了.   

```
seata-v1.4.0
├── all
├── bom
├── changeVersion.sh
├── codecov.yml
├── common                              # 公共模块
├── compressor                          # 压缩
├── config                              # 配置
├── core
├── discovery                           # 服务发现
├── distribution                        # 编译后产生的二进制文件
├── integration
├── metrics                             # 监控
├── pom.xml
├── rm                                 
├── rm-datasource
├── saga
├── script
├── seata-spring-boot-starter
├── serializer                         # 序列化
├── server
├── spring
├── sqlparser
├── style
├── tcc
├── test
└── tm
```

### (4). Seata模块关系图
!["Seata模块依赖图"](/assets/seata/imgs/Seata-Mdule-Dependency.jpg)