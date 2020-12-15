---
layout: post
title: 'SpringCloud Gateway Route Head(七)'
date: 2020-12-15
author: 李新
tags: SpringCloudGateway
---

### (1). Route Head
> Head用来断言(Predicate)请求头(Head)   
> HeaderRoutePredicateFactory

### (2). application.yml
```
#端口
server:
  port: 9000

# id : 路由ID,需要做到唯一
# uri : 微服务的地址
# predicates : 断言(判断条件)
# 路由规则(Header): 匹配请求头

# curl -H "token:123456" http://localhost:9000/consumer

spring:
  application:
    name: gateway-server  # 应用名称
  cloud:
    gateway:
      routes:
        - id: test-consumer-service
          uri: "http://localhost:7070/"
          predicates: 
            # 匹配请求头包含:token,并且其值要符合正则表达式"\d+"的请求.
            - Header=token, \d+
```
### (3). 测试
```
# 错误请求(没有设置请求头)
http://localhost:9000/consumer
{"timestamp":"2020-12-15T09:19:10.188+0000","path":"/consumer","status":404,"error":"Not Found","message":null}

# 成功请求(设置请求头)
curl -H "token:1234" http://localhost:9000/consumer
consumer...Hello World!!!
```