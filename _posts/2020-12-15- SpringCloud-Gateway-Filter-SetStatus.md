---
layout: post
title: 'SpringCloud Gateway Filter SetStatus(十五)'
date: 2020-12-15
author: 李新
tags: SpringCloudGateway
---

### (1). Filter SetStatusGatewayFilterFactory
>  SetStatus将请求设置成相应的状态码(404/200...)

### (3). application.yml
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
            # 更改响应状态为:404
            - SetStatus=404
```
### (4). 测试
```
# 大写的I可以查看响应的协议头信息.
curl -I http://localhost:9000/api-gateway/consumer
HTTP/1.1 404 Not Found
Content-Type: text/plain;charset=UTF-8
Date: Wed, 16 Dec 2020 06:28:12 GMT
Content-Length: 0

```