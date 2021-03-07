---
layout: post
title: 'Spring Cloud Gateway Route Method(四)'
date: 2020-12-15
author: 李新
tags: SpringCloudGateway
---

### (1). Route Method
> Method用来断言(Predicate)请求方式(GET/POST/HEAD/DELETE....).   
> MethodRoutePredicateFactory   

### (2). application.yml
```
#端口
server:
  port: 9000

# org.springframework.cloud.gateway.route.Route
# id : 路由ID,需要做到唯一
# uri : 微服务的地址
# predicates : 断言(判断条件)
# 路由规则(GET): 匹配GET/POST/PUT/HEAD...

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
            - Method=GET
```
### (3). 测试
```
# GET请求(正常)
curl -X GET http://localhost:9000/consumer
consumer...Hello World!!!

# POST请求(失败)
curl -X POST http://localhost:9000/consumer
{"timestamp":"2020-12-15T08:35:46.395+0000","path":"/consumer","status":404,"error":"Not Found","message":null}
```