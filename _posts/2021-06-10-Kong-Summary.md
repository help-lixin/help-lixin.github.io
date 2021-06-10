---
layout: post
title: 'Kong 介绍(一)'
date: 2021-06-09
author: 李新
tags:  Kong
---

### (1). Kong介绍
> Kong是一款基于OpenResty(Nginx + Lua模块)编写的高可用、易扩展的API Gateway项目.   
> 一句话总结:把Nginx的配置进行动态化,不需要每次都进行加载(-s reload),并且,提供了不少的插件(WAF/限流/).    
> [Kong官网](https://docs.konghq.com/gateway-oss/2.4.x)  

### (2). Kong关键术语
+ Route:
   - Route字面意思就是路由,通过定义一些规则来匹配客户端的请求每个路由都会关联一个Service.
+ Service:
   - Service,就是我们自己定义的上游服务,可以把它理解成一个微服务的名称.
+ Upstream:
   - 直接理解成Nginx的upstream即可.
+ Target:
   - 理解成Nginx中upstream配置的每一项.
### (3). Kong学习目录
+["PostgreSQL安装(一)"](/2021/06/09/PostgreSQL-Install.html)    
+["Kong 安装(二)"](/2021/06/09/Kong-Install.html)    
+["Konga 安装(三)"](/2021/06/09/Konga-Install.html)     
+["Kong 配置服务案例(四)"](/2021/06/09/Kong-Api.html)   