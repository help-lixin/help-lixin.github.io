---
layout: post
title: 'SpringCloud Gateway Route DateTime(五)'
date: 2020-12-15
author: 李新
tags: SpringCloudGateway
---

### (1). Route DateTime
> DateTime用来断言(Predicate)请求是否在指定的时间:之前,之间,之后.   
> BeforeRoutePredicateFactory   
> BetweenRoutePredicateFactory    
> AfterRoutePredicateFactory     

### (2). application.yml
```
#端口
server:
  port: 9000

# org.springframework.cloud.gateway.route.Route
# id : 路由ID,需要做到唯一
# uri : 微服务的地址
# predicates : 断言(判断条件)
# 路由规则(Before): 判断当前请求的时间,是否在Before指定的时间之前.当前时间小于Before时间,路由才会通过.
# 路由规则(After): 判断当前请求的时间,是否在After指定的时间之后.当前时间大于After时间,路由才会通过.

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
            # - After=2020-11-11T11:11:11.000+08:00[Asia/Shanghai]
            - Before=2020-12-11T11:11:11.000+08:00[Asia/Shanghai]
            # - Before=2020-12-16T11:11:11.000+08:00[Asia/Shanghai]
           
```
### (3). 测试
```

# before测试
# 当前时间(2020-12-15 < 2020-12-11)为:false
# 结果应该是:404
curl http://localhost:9000/consumer
{"timestamp":"2020-12-15T08:56:18.367+0000","path":"/consumer","status":404,"error":"Not Found","message":null}

# before测试
# 当前时间(2020-12-15 < 2020-12-16)为:true
# 结果应该要正常路由.
curl http://localhost:9000/consumer
consumer...Hello World!!!
```