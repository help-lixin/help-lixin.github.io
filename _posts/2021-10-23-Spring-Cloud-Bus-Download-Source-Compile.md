---
layout: post
title: 'Spring Cloud Bus源码下载并编译(二)' 
date: 2021-10-23
author: 李新
tags:  SpringCloudBus
---

### (1). Spring Cloud Bus源码下载
```
# 1. 进入仓库目录
lixin-macbook:~ lixin$ cd ~/GitRepository/

# 2. clone源码
lixin-macbook:GitRepository lixin$ git clone https://github.com/help-lixin/spring-cloud-bus.git

# 3. 进入:spring-cloud-bus目录
lixin-macbook:GitRepository lixin$ cd spring-cloud-bus/

# 4. 好像master分支我编译不通过,切换到3.0x分支
lixin-macbook:spring-cloud-bus lixin$ git checkout 3.0.x

# 5. 编译
lixin-macbook:spring-cloud-bus lixin$ mvn clean install -DskipTests
```
### (2). Spring Cloud Bus源码目录结构
```
lixin-macbook:spring-cloud-bus lixin$ tree -L 1
.
├── pom.xml
├── spring-cloud-bus                               # 核心代码
├── spring-cloud-bus-dependencies                  # 依赖
├── spring-cloud-bus-rsocket                       # rsocket结合
├── spring-cloud-bus-tests                         # 测试代码
├── spring-cloud-starter-bus-amqp                  # amqp(rabbitmq)
├── spring-cloud-starter-bus-kafka                 # kafka
├── spring-cloud-starter-bus-stream                # spring-cloud-stream
├── src
└── target
```
### (3). 切入点在哪?
> 答案就在: spring.factories 

```
lixin-macbook:spring-cloud-bus lixin$ cat ./spring-cloud-bus/src/main/resources/META-INF/spring.factories 
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.bus.BusPropertiesAutoConfiguration,\
org.springframework.cloud.bus.BusAutoConfiguration,\
org.springframework.cloud.bus.jackson.BusJacksonAutoConfiguration

# Environment Post Processor
org.springframework.boot.env.EnvironmentPostProcessor=\
org.springframework.cloud.bus.BusEnvironmentPostProcessor
```
### (4). 总结
Spring Cloud Bus的代码比较简单,大概就几十个类左右,后面,会从spring.factories开始分析源码.  