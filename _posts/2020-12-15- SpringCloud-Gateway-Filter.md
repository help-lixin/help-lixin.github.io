---
layout: post
title: 'SpringCloud Gateway Filter(十)'
date: 2020-12-15
author: 李新
tags: SpringCloudGateway
---

### (1). Eureka 结合 Route
> pom.xml添加eureka.

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
  </dependency>
```
### (2). application.yml
```
#端口
server:
  port: 9000

# 配置eureka注册中心
eureka:
  instance:
    prefer-ip-address: true
    instance-id: ${spring.cloud.client.ip-address}:${server.port}
  client:
    service-url:
        defaultZone: http://eureka1:1111/eureka/,http://eureka1:2222/eureka/

spring:
  application:
    name: gateway-server  # 应用名称
  cloud:
    gateway:
      discovery:
          locator:
              # 是否与服务发现组件进行结合,通过serviceId转发到具体服务实例上.
              enabled: true                  # 是否开启基于服务发现的路由规则
              lower-case-service-id: true    # 是否将服务名称小写
```
### (4). 测试
```
# 访问(test-consumer-feign)服务
curl  http://localhost:9000/test-consumer-feign/consumer
consumer...Hello World!!!


# 访问(test-provider)服务
curl  http://localhost:9000/test-provider/hello
Hello World!!!
```