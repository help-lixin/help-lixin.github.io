---
layout: post
title: 'Servicecomb Pack之Saga本地环境搭建入门(三)'
date: 2021-05-04
author: 李新
tags:  Servicecomb-Pack
---

### (1). 前言
> 前面,通过Docker+Servicecomb Pack,实现简单的入门,但是,要想实现远程调试,部署到Docker没有本地那么方便,在这一小节,还是上面的项目,只是,不再用Docker运行,方便后续进行Debug.  

### (2). alpha-server改造
```
# 更换成mysql(/Users/lixin/GitRepository/servicecomb-pack-0.6.0/alpha/alpha-server/pom.xml)
# 1. 添加mysql驱动
<dependency>
  <groupId>mysql</groupId>
  <artifactId>mysql-connector-java</artifactId>
  <version>5.1.49</version>
</dependency> 
 
 
# 2. 禁用postgresql
<!--
 <dependency>
   <groupId>org.postgresql</groupId>
   <artifactId>postgresql</artifactId>
   <scope>runtime</scope>
 </dependency>
-->


# 3. 配置application.yml(/Users/lixin/GitRepository/servicecomb-pack-0.6.0/alpha/alpha-server/src/main/resources/application.yaml)
#    配置:driverClassName/username/password/url
spring:
  profiles: mysql
  datasource:
    driverClassName: com.mysql.jdbc.Driver
    username: root
    password: 123456
    url: jdbc:mysql://127.0.0.1:3306/saga?useSSL=false
    platform: mysql
    continue-on-error: false
  jpa:
    properties:
      eclipselink:
        ddl-generation: none

# 4. 进入MySQL,创建:saga库和表
CREATE DATABASE saga;
USE saga;
SOURCE /Users/lixin/GitRepository/servicecomb-pack-0.6.0/alpha/alpha-server/src/main/resources/schema-mysql.sql
```
### (3). alpha-server启动
> alpha-server启动,并指定环境变量(java -Dspring.profiles.active=mysql -jar alpha-server.jar).

!["AlphaApplication启动,并指定环境变量"](/assets/servicecomb-pack/imgs/AlphaApplication.jpg)

### (4). hotel
> /Users/lixin/GitRepository/servicecomb-pack-0.6.0/demo/saga-spring-demo/hotel      
> hotel启动,并指定环境变量(java -Dserver.port=8081  -Dalpha.cluster.address=localhost:8080 -jar hotel.jar )

!["hotel启动,并指定环境变量"](/assets/servicecomb-pack/imgs/hotel.jpg)

### (5). car 
> /Users/lixin/GitRepository/servicecomb-pack-0.6.0/demo/saga-spring-demo/car     
> car启动,并指定环境变量(java -Dserver.port=8082  -Dalpha.cluster.address=localhost:8080 -jar car.jar )

!["car启动,并指定环境变量"](/assets/servicecomb-pack/imgs/car.jpg)

### (6). booking
> /Users/lixin/GitRepository/servicecomb-pack-0.6.0/demo/saga-spring-demo/booking    
> booking启动,并指定环境变量(java -Dserver.port=8083  -Dalpha.cluster.address=localhost:8080 -Dhotel.service.address=http://localhost:8081  -Dcar.service.address=http://localhost:8082  -jar booking.jar )

!["booking启动,并指定环境变量"](/assets/servicecomb-pack/imgs/booking.jpg)

### (7). 测试验证
```
# 模拟预订车辆和酒店房间
lixin-macbook:5.1.49 lixin$ curl -X POST http://localhost:8083/booking/test/2/2
test booking 2 rooms and 2 cars OK

# 查看服务状态
lixin-macbook:5.1.49 lixin$ curl http://localhost:8081/bookings
[{"id":1,"name":"test","amount":2,"confirmed":true,"cancelled":false}]
```
### (8).  总结
> 后续会基于这个案例的基础上,对Saga进行深入的源码探索.  