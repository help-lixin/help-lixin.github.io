---
layout: post
title: 'Spring Cloud Gateway Filter SetPath(十三)'
date: 2020-12-15
author: 李新
tags: SpringCloudGateway
---

### (1). Filter SetPathGatewayFilterFactory
>  SetPath它允许模板化路径段来操作请求路径的简单写法

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
            - Path=/api/consumer/{segment}
          filters:
            # 将/api/consumer/Welcome 重写为: /consumer/Welcome
            - SetPath=/consumer/{segment}
```
### (3). 测试
```
# 将/api/consumer/Welcome  重写为: /consumer/Welcome
curl http://localhost:9000/api/consumer/Welcome
consumer...Welcome Hello World!!!
```