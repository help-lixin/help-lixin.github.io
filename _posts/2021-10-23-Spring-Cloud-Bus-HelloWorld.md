---
layout: post
title: 'Spring Cloud Bus简单入门案例(一)' 
date: 2021-10-23
author: 李新
tags:  SpringCloudBus
---

### (1). 概述
在这一小篇,我们搭建一个Spring Cloud Bus的["案例"](/assets/spring-cloud-bus/spring-cloud-bus-parent-example.zip)来入门.

### (2). RabbitMQ搭建(略)


### (3). 项目结构如下图
+ spring-cloud-bus-server修改配置,并且发布刷新事件.       
+ spring-cloud-bus-application热更新配置信息.   

```
lixin-macbook:spring-cloud-bus-parent-example lixin$ tree 
.
├── pom.xml
├── spring-cloud-bus-application                     # 应用程序
│   ├── pom.xml
│   └── src
│       ├── main
│       │   ├── java
│       │   │   └── help
│       │   │       └── lixin
│       │   │           └── bus
│       │   │               ├── Application.java
│       │   │               ├── controller
│       │   │               │   └── HelloController.java
│       │   │               └── model
│       │   │                   └── User.java
│       │   └── resources
│       │       └── application.yml
│       └── test
│           └── java
├── spring-cloud-bus-server                          # bus服务端
│   ├── pom.xml
│   └── src
│       ├── main
│       │   ├── java
│       │   │   └── help
│       │   │       └── lixin
│       │   │           └── bus
│       │   │               ├── BusApplication.java
│       │   │               └── controller
│       │   │                   └── HelloController.java
│       │   └── resources
│       │       └── application.yml
│       └── test
│           └── java
└── src
    ├── main
    │   ├── java
    │   └── resources
    └── test
        └── java
```
### (4). 查看应用程序的配置信息
```
#  8080为:spring-cloud-bus-server
# 1. 在未刷新配置之前,先查看请求内容
lixin-macbook:~ lixin$ curl http://localhost:8080/user
{"name":"lixin","age":25}
```
### (5). 修改配置
```
# 7070为:spring-cloud-bus-application

# 修改配置文件事件:
# user.name = Sun
lixin-macbook:~ lixin$ curl -H 'Content-Type: application/json' -d '{"name":"user.name","value":"Sun"}'   http://localhost:7070/actuator/bus-env

# 刷新事件
# lixin-macbook:~ lixin$ curl -X POST http://localhost:7070/actuator/bus-refresh
```
### (6). 验证应用程序配置信息
```
lixin-macbook:~ lixin$ curl http://localhost:8080/user
{"name":"Sun","age":25}
```
### (7). 总结
向spring-cloud-bus-application发送请求,就可以,所有应用程序配置的热更新.   