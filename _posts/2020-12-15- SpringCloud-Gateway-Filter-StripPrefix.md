---
layout: post
title: 'Spring Cloud Gateway Filter StripPrefix(十二)'
date: 2020-12-15
author: 李新
tags: SpringCloudGateway
---

### (1). Filter StripPrefixGatewayFilterFactory
>  StripPrefix网关过滤器工厂采用一个参数StripPrefix,该参数表示在将请求发送给下游之前剥离的路径个数.

### (2). application.yml
```
#端口
server:
  port: 9000

# id : 路由ID,需要做到唯一
# uri : 微服务的地址
# predicates : 断言(判断条件)
# filter: 把/test/index/consumer   重写为:  /consumer
# 路由规则(Path): 匹配URL的请求,将匹配的请求追加在目标URI之后

spring:
  application:
    name: gateway-server  # 应用名称
  cloud:
    gateway:
      routes:
        - id: test-consumer-service
          uri: "http://localhost:7070/"
          predicates: 
            # 匹配URL的请求,将匹配的请求追加在目标URI之后
            - Path=/**
          filters:
            # 把/test/index/consumer   重写为:  /consumer
            - StripPrefix=2
```
### (3). 测试
```
# 实际处理后的请求是:http://localhost:9000/consumer
curl http://localhost:9000/test/index/consumer
consumer...Hello World!!!
```