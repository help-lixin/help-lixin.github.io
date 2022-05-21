---
layout: post
title: 'ElasticSearch源码编译本地调试' 
date: 2022-05-15
author: 李新
tags:  ElasticSearch源码
---

### (1). 概述
前面学习ElasticSearch源码时,都是远程调试,相当的不方便,一直都想切换到本地调试模式,踩了很多的坑,特意记录下. 

### (2). 前置条件
```
1. es源码分支为: origin/7.10
2. jdk版本为: 11
```
### (3). 源码编译
```
./gradlew :distribution:archives:darwin-tar:assemble
```
### (4). 配置policy
```
lixin-macbook:config lixin$ cat /Users/lixin/Developer/elasticsearch-7.10.3/config/policy.policy
grant {
	permission java.net.SocketPermission "*:*","connect,resolve";
	permission java.lang.RuntimePermission "getClassLoader";
	permission java.lang.RuntimePermission "createClassLoader";
	permission java.lang.RuntimePermission "setContextClassLoader";
};
```
### (5). IDEA配置VM_OPTIONS
!["IDEA配置dependencies with provider scope to classpath"](/assets/elasticsearch/imgs/es-local-debug-2.png)

!["IDEA配置vm_options"](/assets/elasticsearch/imgs/es-debug.png)

```
-Xshare:auto
-Des.networkaddress.cache.ttl=60
-Des.networkaddress.cache.negative.ttl=10
-XX:+AlwaysPreTouch
-Xss1m
-Djava.awt.headless=true
-Dfile.encoding=UTF-8
-Djna.nosys=true
-XX:-OmitStackTraceInFastThrow
-Dio.netty.noUnsafe=true
-Dio.netty.noKeySetOptimization=true
-Dio.netty.recycler.maxCapacityPerThread=0
-Dio.netty.allocator.numDirectArenas=0
-Dlog4j.shutdownHookEnabled=false
-Dlog4j2.disable.jmx=true
-Djava.locale.providers=SPI,COMPAT
-Xms1g
-Xmx1g
-XX:+UseConcMarkSweepGC
-XX:CMSInitiatingOccupancyFraction=75
-XX:+UseCMSInitiatingOccupancyOnly
-Des.path.home=/Users/lixin/Developer/elasticsearch-7.10.3
-Des.path.conf=/Users/lixin/Developer/elasticsearch-7.10.3/config
-Des.distribution.flavor=default
-Des.distribution.type=tar
-Des.bundled_jdk=true
-Djava.security.policy=/Users/lixin/Developer/elasticsearch-7.10.3/config/policy.policy
```
### (6). 总结
