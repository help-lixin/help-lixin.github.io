---
layout: post
title: 'Spring Cloud Gateway Route RemoteAddr(六)'
date: 2020-12-15
author: 李新
tags: SpringCloudGateway
---

### (1). Route RemoteAddr
> RemoteAddr用来断言(Predicate)请求的IP地址.   
> RemoteAddrRoutePredicateFactory

### (2). application.yml
```
#端口
server:
  port: 9000

# org.springframework.cloud.gateway.route.Route
# id : 路由ID,需要做到唯一
# uri : 微服务的地址
# predicates : 断言(判断条件)
# 路由规则(RemoteAddr): 匹配请求的IP地址

#http://localhost:9000/consumer

spring:
  application:
    name: gateway-server  # 应用名称
  cloud:
    gateway:
      routes:
        - id: test-consumer-service
          uri: "http://localhost:7070/"
          predicates: 
            - RemoteAddr=172.17.6.1/0    #匹配远程地址请求是:RemoteAddr的请求,0表示子网掩码.
```
### (3). 测试
```
# 正确访问:
curl http://172.17.6.74:9000/consumer
consumer...Hello World!!!

# 错误访问:
curl http://localhost:9000/consumer
{"timestamp":"2020-12-15T09:09:40.530+0000","path":"/consumer","status":404,"error":"Not Found","message":null}
```