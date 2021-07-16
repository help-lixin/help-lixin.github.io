---
layout: post
title: 'Apollo架构深入浅出(一)'
date: 2021-07-16
author: 李新
tags:  Apollo
---

### (1). Apollo是什么
Apollo(阿波罗)是携程框架部门研发的分布式配置中心,能够集中化管理应用不同环境、不同集群的配置,配置修改后能够实时推送到应用端,并且具备规范的权限、流程治理等特性,适用于微服务配置管理场景.   
服务端基于Spring Boot和Spring Cloud开发,打包后可以直接运行,不需要额外安装Tomcat等应用容器. 
Java客户端不依赖任何框架,能够运行于所有Java运行时环境,同时对Spring/Spring Boot环境也有较好的支持.
.Net客户端不依赖任何框架,能够运行于所有.Net运行时环境.

### (2). 基本概念
+ application(应用)
  - 这个很好理解,就是实际要使用Apollo配置的应用程序(比如:order-service),Apollo客户端在运行时需要知道当前应用是谁,从而可以去获取对应的配置.
  - 每个应用都需要有唯一的身份标识(appId),我们认为应用身份是跟着代码走的,所以需要在代码中配置.
+ environment(环境)
  - 配置对应的环境(开发环境/测试环境/生产环境),Apollo客户端在运行时需要知道当前应用处于哪个环境,从而可以去获取应用的配置我们认为环境和代码无关,同一份代码部署在不同的环境就应该能够获取到不同环境的配置.
  - 环境默认是通过读取机器上的配置(server.properties中的env属性)指定的,不过为了开发方便,也支持运行时通过System Property指定.  
+ cluster(集群)
  - 一个应用下不同实例的分组,比如典型的可以按照数据中心分,把上海机房的应用实例分为一个集群,把北京机房的应用实例分为另一个集群.
  - 对不同的cluster,同一个配置可以有不一样的值,如zookeeper地址.
  - 集群默认是通过读取机器上的配置(server.properties中的idc属性)指定的,不过也支持运行时通过System Property指定.
+ namespace(命名空间)
  - 一个应用下不同配置的分组,可以简单地把namespace类比为文件,不同类型的配置存放在不同的文件中,如数据库配置文件,RPC配置文件,应用自身的配置文件等.
  - 应用可以直接读取到公共组件的配置namespace，如DAL，RPC等.
  - 应用也可以通过继承公共组件的配置namespace来对公共组件的配置做调整，如DAL的初始数据库连接数

### (3). Apollo架构设计
下图简要描述了Apollo的总体设计,我们可以从下往上看:
+ Config Service提供配置的读取、推送等功能,服务对象是Apollo客户端.
+ Admin Service提供配置的修改、发布等功能,服务对象是Apollo Portal(管理界面).  
+ Config Service和Admin Service都是多实例、无状态部署,所以需要将自己注册到Eureka中并保持心跳.  
+ 在Eureka之上架了一层Meta Server用于封装Eureka的服务发现接口
+ Client通过域名访问Meta Server获取Config Service服务列表(IP+Port),而后直接通过IP+Port访问服务,同时在Client侧会做load balance、错误重试.    
+ Portal通过域名访问Meta Server获取Admin Service服务列表(IP+Port),而后直接通过IP+Port访问服务,同时在Portal侧会做load balance、错误重试.   
+ 为了简化部署,实际上会把Config Service、Eureka和Meta Server三个逻辑角色部署在同一个JVM进程中.   


!["Apollo总体架构"](/assets/apollo/imgs/apollo-overall-architecture.png)

### (4). Apollo客户端设计
下图简要描述了Apollo客户端的实现原理:
+ 客户端和服务端保持了一个长连接,从而能第一时间获得配置更新的推送.  
+ 客户端还会定时从Apollo配置中心服务端拉取应用的最新配置.
  - 这一步是为了防止推送机制失效导致配置不更新,所以,定时任务去拉取配置.  
  - 客户端定时拉取会上报本地版本,所以一般情况下,对于定时拉取的操作,服务端都会返回304 - Not Modified.  
  - 定时频率默认为每5分钟拉取一次,客户端也可以通过在运行时指定System Property: apollo.refreshInterval来覆盖(单位为分钟)
+ 客户端从Apollo配置中心服务端获取到应用的最新配置后,会保存在内存中.  
+ 客户端会把从服务端获取到的配置在本地文件系统缓存一份.
  - 在遇到服务不可用,或网络不通的时候,依然能从本地恢复配置.
+ 应用程序从Apollo客户端获取最新的配置、订阅配置更新通知

!["Apollo客户端设计"](/assets/apollo/imgs/apollo-client-architecture.png)

### (5). 配置更新推送实现细节
Apollo客户端和服务端保持了一个长连接,从而能第一时间获得配置更新的推送.  
长连接实际上是通过Http Long Polling实现的,具体做法如下:
+ 客户端发起一个Http请求到服务端
+ 服务端会保持住(Hold)这个连接60秒.
  - 如果在60秒内有客户端关心的配置变化,被保持住的客户端请求会立即返回,并告知客户端有配置变化的namespace信息,客户端会据此拉取对应namespace的最新配置(注意:推只是事件,底层还是要靠拉取),为什么这样设计?因为,如果client太多,要推送的内容以及次数太多,会消耗服务器资源.    
  - 如果在60秒内没有客户端关心的配置变化,那么会返回Http状态码304给客户端.
+ 客户端在收到服务端请求后会立即重新发起连接,回到第一步

### (6). 总结
Apollo在设计上,考虑还是挺全面的.后面会进行项目搭建以及源码剖析.  