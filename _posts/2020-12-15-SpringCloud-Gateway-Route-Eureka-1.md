---
layout: post
title: 'Spring Cloud Gateway Route结合Eureka-1(八)'
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

# id : 路由ID,需要做到唯一
# uri : lb://根据微服务的名称从注册中心获取请求地址.
# predicates : 断言(判断条件)
# 路由规则(Path): 根据URL进行匹配,并把URL加在lb的后面
# 假设配置:Path=/test/consumer/**,那么,请求的URL则是:(http://test-consumer-feign/test/consumer/**)

spring:
  application:
    name: gateway-server  # 应用名称
  cloud:
    gateway:
      routes:
        - id: test-consumer-service
          uri: lb://test-consumer-feign
          predicates: 
            - Path=/consumer/**
        - id: test-provider-service
          uri: lb://test-provider
          predicates: 
            - Path=/hello/**
```
### (3). 查看Eureka状态
!["Spring Cloud Gateway注册到Eureka,并获取Eureka的路由信息"](/assets/spring-cloud-gateway/imgs/spring-cloud-gateway-uri-lb.jpg)

### (4). 测试
```
# uri配置的是lb
# spring cloud gateway会根据服务名称(test-consumer-feign)转换成:ip地址访问.

curl  http://localhost:9000/consumer
consumer...Hello World!!!
```