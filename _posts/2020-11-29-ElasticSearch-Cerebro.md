---
layout: post
title: 'ElasticSearch Cerebro下载和安装(三)'
date: 2020-11-29
author: 李新
tags: ElasticSearch
---

### (1). 下载Cerebro插件

> wget https://github.com/lmenezes/cerebro/releases/download/v0.9.2/cerebro-0.9.2.tgz   

### (2). 配置ES集群信息
> /conf/application.conf

```
hosts = [
  {
    host = "http://localhost:9200"
    name = "localhost-9200"
  }
]
```

### (3). 启动
```
# 启动并指定端口
lixin-macbook:elastic-search lixin$ ./cerebro-0.9.2/bin/cerebro -Dhttp.port=9001

WARNING: An illegal reflective access operation has occurred
WARNING: Illegal reflective access by com.google.inject.internal.cglib.core.$ReflectUtils$1 (file:/Users/lixin/Developer/elastic-search/cerebro-0.9.2/lib/com.google.inject.guice-4.2.2.jar) to method java.lang.ClassLoader.defineClass(java.lang.String,byte[],int,int,java.security.ProtectionDomain)
WARNING: Please consider reporting this to the maintainers of com.google.inject.internal.cglib.core.$ReflectUtils$1
WARNING: Use --illegal-access=warn to enable warnings of further illegal reflective access operations
WARNING: All illegal access operations will be denied in a future release
[info] play.api.Play - Application started (Prod) (no global state)
# 监听端口成功
[info] p.c.s.AkkaHttpServer - Listening for HTTP on /0:0:0:0:0:0:0:0:9001
```