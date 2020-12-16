---
layout: post
title: 'SpringCloud Gateway Filter PrefixPath(十一)'
date: 2020-12-15
author: 李新
tags: SpringCloudGateway
---

### (1). Filter PrefixPathGatewayFilterFactory
>  PrefixPath为匹配URL添加指定的前缀.

### (3). application.yml
```
#端口
server:
  port: 9000

# org.springframework.cloud.gateway.route.Route
# id : 路由ID,需要做到唯一
# uri : 微服务的地址
# predicates : 断言(判断条件)
# filters: 请求:/hello 会增加前缀变成:/consumer/hello
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
            # 请求:/hello 会增加前缀变成:/consumer/hello
            - PrefixPath=/consumer
```
### (4). 测试
```
# 访问:http://localhost:9000/
# 会为这个请求增加前缀:http://localhost:7070/consumer/
curl http://localhost:9000/
consumer...Hello World!!!

# 访问:http://localhost:9000/hello
# 会为这个请求增加前缀:http://localhost:7070/consumer/hello
curl http://localhost:9000/hello
{  "timestamp":"2020-12-16T05:36:27.590+0000",
   "status":404,
   "error":"Not Found",
   "message":"No message available",
   "path":"/consumer/hello"
}
```