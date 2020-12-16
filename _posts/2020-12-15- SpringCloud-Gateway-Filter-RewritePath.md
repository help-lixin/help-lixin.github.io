---
layout: post
title: 'Spring Cloud Gateway Filter RewritePath(十)'
date: 2020-12-15
author: 李新
tags: SpringCloudGateway
---

### (1). Filter RewritePathGatewayFilterFactory
>  Path路径过滤器可以实现URL重写,通过重写URL可以实现隐藏实际路径提高安全性.

### (2). RewritePathGatewayFilterFactory
> RewritePath网关过滤器工厂采用路径正则表达式参数和替换参数.使用Java正则表达式灵活地重写请求路径. 

### (3). application.yml
```
#端口
server:
  port: 9000

# id : 路由ID,需要做到唯一
# uri : 微服务的地址
# predicates : 断言(判断条件)
# filters:把/api-gateway/consumer 重写成 /consumer
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
            # - Path=/consumer/** , /api-gateway/**
            - Path=/api-gateway/**
          filters:
            # 把/api-gateway/consumer 重写成 /consumer
            - RewritePath=/api-gateway(?<segment>/?.*), $\{segment}
```
### (4). 测试
```
# 请求:http://localhost:9000/api-gateway/consumer地址会被改写成:
# http://localhost:7070/consumer
curl http://localhost:9000/api-gateway/consumer
consumer...Hello World!!!
```