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
├── config                              # 配置(可与Nacos/etcd/apollo/zk整合)
├── core                                # 核心模块
├── discovery                           # 与注册中心的接入和负载均衡的功能
├── distribution                        # 编译后产生的二进制文件
├── integration
├── metrics                             # 统计模块
├── pom.xml
├── tm                                  # 对 TM 的实现,提供了全局事务管理,例如:事务的发起,提交,回滚
├── rm                                  # Resouce Manager
├── rm-datasource                       # Resouce Mnager与DataSouce的切入
├── saga                                # saga模块
├── script
├── seata-spring-boot-starter
├── serializer                         # 序列化
├── server
├── spring
├── sqlparser
├── style
├── test
└── tcc                                   # Seata 对 TCC 事务模式的实现
```

### (4). Seata模块关系图
!["Seata模块依赖图"](/assets/seata/imgs/Seata-Mdule-Dependency.jpg)