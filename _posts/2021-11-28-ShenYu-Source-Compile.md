---
layout: post
title: 'ShenYu源码下载与编译(一)' 
date: 2021-11-28
author: 李新
tags:  ShenYu
---

### (1). Apache ShenYu介绍
Apache ShenYu是一个基于Gateway研发的异步的,高性能的,跨语言的,响应式的API网关.    

- 支持各种语言(http 协议),支持 Dubbo、 Spring Cloud、 gRPC、 Motan、 Sofa、 Tars等协议.   
- 插件化设计思想,插件热插拔，易扩展.   
- 灵活的流量筛选,能满足各种流量控制.   
- 内置丰富的插件支持,鉴权,限流,熔断,防火墙等等.   
- 流量配置动态化,性能极高.   
- 支持集群部署,支持 A/B Test,蓝绿发布.   
### (2). Apache ShenYu架构图
- client(消费者)   
- admin(管理界面)    
- gateway cluster(网关)     
- dubbo cluser/http request/web service(提供者)  

!["ShenYu架构图"](/assets/shenyu/imgs/shenyu-framework-e166860ec2167748123b536ea91d312e.png)   
### (3). 源码下载并编译
```
lixin-macbook:~ lixin$ cd ~/GitRepository/
lixin-macbook:GitRepository lixin$ git clone https://github.com/help-lixin/incubator-shenyu.git
lixin-macbook:GitRepository lixin$ cd incubator-shenyu/
lixin-macbook:GitRepository lixin$ mvn clean install -DskipTests -X
```
### (4). admin启动

```
# 修改application.yml

spring:
  profiles:
    active: mysql
	
sync:
#    websocket:
#      enabled: true
      zookeeper:
        url: localhost:2181
        sessionTimeout: 5000
        connectionTimeout: 2000
	

# 修改application-mysql.yml
shenyu:
  database:
    dialect: mysql
    init_script: "sql-script/mysql/schema.sql"
    init_enable: true
    
spring:
  datasource:
    url: jdbc:mysql://127.0.0.1:3306/shenyu?useUnicode=true&characterEncoding=utf-8&useSSL=false
    username: root
    password: 123456
    driver-class-name: com.mysql.jdbc.Driver

# 添加依赖
<dependency>
	<groupId>com.alibaba</groupId>
	<artifactId>druid</artifactId>
</dependency>

# 注释掉provided
<dependency>
	<groupId>mysql</groupId>
	<artifactId>mysql-connector-java</artifactId>
	<version>${mysql.version}</version>
	<!-- <scope>provided</scope>  -->
</dependency>
```

### (5). 启动admin
> 账号和密码为: admin/123456   

!["admin登录界面"](/assets/shenyu/imgs/shenyu-admin.png)

### (6). 源码目录介绍
```
lixin-macbook:incubator-shenyu lixin$ tree -L 2
.
├── pom.xml
├── script              # 脚本
│   ├── 2.4.1-upgrade-2.4.2.sql
│   ├── shenyu_checkstyle.xml
│   └── upgrade-guide.md
├── shenyu-admin        # admin
│   ├── README.md
│   ├── pom.xml
│   ├── src
│   └── target
├── shenyu-agent       # 代理
│   ├── pom.xml
│   ├── shenyu-agent-api
│   ├── shenyu-agent-bootstrap
│   ├── shenyu-agent-core
│   ├── shenyu-agent-plugins
│   └── target
├── shenyu-bootstrap  # 引导器
│   ├── pom.xml
│   ├── src
│   └── target
├── shenyu-client     # 服务提供者需要整合使用的客户端.
│   ├── pom.xml
│   ├── shenyu-client-core
│   ├── shenyu-client-dubbo
│   ├── shenyu-client-grpc
│   ├── shenyu-client-http
│   ├── shenyu-client-motan
│   ├── shenyu-client-sofa
│   ├── shenyu-client-tars
│   └── target
├── shenyu-common     # 框架的通用类
│   ├── pom.xml
│   ├── src
│   └── target
├── shenyu-disruptor # 基于disruptor的封装
│   ├── pom.xml
│   ├── src
│   └── target
├── shenyu-dist      # 二进制分发包
│   ├── README.md
│   ├── pom.xml
│   ├── release_check.py
│   ├── shenyu-admin-dist
│   ├── shenyu-bootstrap-dist
│   ├── shenyu-docker-compose-dist
│   ├── shenyu-src-dist
│   └── target
├── shenyu-examples   # 演示案例
│   ├── pom.xml
│   ├── shenyu-examples-dubbo
│   ├── shenyu-examples-eureka
│   ├── shenyu-examples-grpc
│   ├── shenyu-examples-http
│   ├── shenyu-examples-motan
│   ├── shenyu-examples-plugin
│   ├── shenyu-examples-sofa
│   ├── shenyu-examples-springcloud
│   ├── shenyu-examples-tars
│   └── shenyu-examples-websocket
├── shenyu-integrated-test  # 集成测试代码
│   ├── README.md
│   ├── pom.xml
│   ├── shenyu-integrated-test-combination
│   ├── shenyu-integrated-test-common
│   ├── shenyu-integrated-test-grpc
│   ├── shenyu-integrated-test-http
│   ├── shenyu-integrated-test-motan
│   ├── shenyu-integrated-test-sofa
│   ├── shenyu-integrated-test-spring-cloud
│   ├── shenyu-integrated-test-websocket
│   ├── shenyu-integration-test-alibaba-dubbo
│   └── shenyu-integration-test-apache-dubbo
├── shenyu-loadbalancer    # 负载均衡
│   ├── pom.xml
│   └── src
├── shenyu-metrics         # 指标监控
│   ├── pom.xml
│   ├── shenyu-metrics-facade
│   ├── shenyu-metrics-prometheus
│   └── shenyu-metrics-spi
├── shenyu-plugin          # shenyu所有的插件
│   ├── pom.xml
│   ├── shenyu-plugin-api
│   ├── shenyu-plugin-base
│   ├── shenyu-plugin-context-path
│   ├── shenyu-plugin-cryptor
│   ├── shenyu-plugin-divide
│   ├── shenyu-plugin-dubbo
│   ├── shenyu-plugin-global
│   ├── shenyu-plugin-grpc
│   ├── shenyu-plugin-httpclient
│   ├── shenyu-plugin-hystrix
│   ├── shenyu-plugin-jwt
│   ├── shenyu-plugin-logging
│   ├── shenyu-plugin-modify-response
│   ├── shenyu-plugin-monitor
│   ├── shenyu-plugin-motan
│   ├── shenyu-plugin-oauth2
│   ├── shenyu-plugin-param-mapping
│   ├── shenyu-plugin-ratelimiter
│   ├── shenyu-plugin-redirect
│   ├── shenyu-plugin-request
│   ├── shenyu-plugin-resilience4j
│   ├── shenyu-plugin-response
│   ├── shenyu-plugin-rewrite
│   ├── shenyu-plugin-sentinel
│   ├── shenyu-plugin-sign
│   ├── shenyu-plugin-sofa
│   ├── shenyu-plugin-springcloud
│   ├── shenyu-plugin-tars
│   ├── shenyu-plugin-uri
│   ├── shenyu-plugin-waf
│   ├── shenyu-plugin-websocket
│   └── target
├── shenyu-protocol         # 协议支持 
│   ├── pom.xml
│   ├── shenyu-protocol-grpc
│   ├── shenyu-protocol-mqtt
│   └── target
├── shenyu-register-center   # 注册中心
│   ├── pom.xml
│   ├── shenyu-register-client
│   ├── shenyu-register-common
│   ├── shenyu-register-instance
│   └── shenyu-register-server
├── shenyu-spi               # spi
│   ├── pom.xmlkh 
│   ├── src
│   └── target
├── shenyu-spring-boot-starter   # boot starter
│   ├── pom.xml
│   ├── shenyu-spring-boot-starter-client
│   ├── shenyu-spring-boot-starter-gateway
│   ├── shenyu-spring-boot-starter-instance
│   ├── shenyu-spring-boot-starter-plugin
│   ├── shenyu-spring-boot-starter-sync-data-center
│   └── target
├── shenyu-sync-data-center      # 数据同步中心
│   ├── pom.xml
│   ├── shenyu-sync-data-api
│   ├── shenyu-sync-data-consul
│   ├── shenyu-sync-data-etcd
│   ├── shenyu-sync-data-http
│   ├── shenyu-sync-data-nacos
│   ├── shenyu-sync-data-websocket
│   └── shenyu-sync-data-zookeeper
├── shenyu-web                    # web
│   ├── pom.xml
│   ├── src
│   └── target
```