---
layout: post
title: 'Spring Cloud Gateway Route Query(三)'
date: 2020-12-15
author: 李新
tags: SpringCloudGateway
---

### (1). Route Query
> Query用来断言(Predicate)请求的参数.   
> QueryRoutePredicateFactory   

### (2). application.yml
```
#端口
server:
  port: 9000

# id : 路由ID,需要做到唯一
# uri : 微服务的地址
# predicates : 断言(判断条件)
# 路由规则(Query): 匹配请求的参数中包含"token",并且参数值满足正则表达式"abc."的请求."."代表任意一个字符.

# 正确请求:http://localhost:9000/consumer?token=abc1
# 错误请求:http://localhost:9000/consumer?token=abc11

spring:
  application:
    name: gateway-server  # 应用名称
  cloud:
    gateway:
      routes:
        - id: test-consumer-service
          uri: "http://localhost:7070/"
          predicates: 
            - Query=token, abc.    #正则请求(?token=abc)
            # - Query=token    #请求参数中包含有:token(?token=)
      
```
### (3). 测试
> <font color='red'>注意:"abc.",中的点,代表任意一个字符,不是任意多个字符.</font>

```
# 正确请求示例:
curl -X GET http://localhost:9000/consumer?token=abc1
consumer...Hello World!!!

# 错误请求示例:
curl -X GET http://localhost:9000/consumer?token=abc12
{"timestamp":"2020-12-15T08:38:45.614+0000","path":"/consumer","status":404,"error":"Not Found","message":null}
```