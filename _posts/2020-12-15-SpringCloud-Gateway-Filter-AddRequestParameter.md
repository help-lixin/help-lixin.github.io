---
layout: post
title: 'Spring Cloud Gateway Filter AddRequestParameter(十四)'
date: 2020-12-15
author: 李新
tags: SpringCloudGateway
---

### (1). Filter AddRequestParameterGatewayFilterFactory
>  可以在请求发往下游的时候,增加参数

### (2). application.yml
```
#端口
server:
  port: 9000

# id : 路由ID,需要做到唯一
# uri : 微服务的地址
# predicates : 断言(判断条件)
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
            - Path=/api-gateway/**
          filters:
            # 将/api-gateway/consumer 重写成 /consumer
            - RewritePath=/api-gateway(?<segment>/?.*), $\{segment}
            # 在下游请示中增加参数: /consumer?token=abc123
            - AddRequestParameter=token,abc123
```
### (3). 测试
```
# 可以在consumer打印URL信息.
# 最终的请求是:http://localhost:7070/consumer?token=abc123
curl http://localhost:9000/api-gateway/consumer
consumer...Hello World!!!
```