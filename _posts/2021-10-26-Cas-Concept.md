---
layout: post
title: 'CAS基本概念介绍(一)' 
date: 2021-10-26
author: 李新
tags:  CAS
---

### (1). SSO是什么
SSO是英文Single Sign On的缩写,翻译过来就是单点登录.它是目前比较流行的服务于"企业业务整合"的解决方案之一,SSO使得在多个应用系统中,用户只需要登录一次就可以访问所有"相互信任的应用系统".
### (2). CAS是什么
CAS(Central Authentication Service)是Yale大学发起的一个企业级的、开源的项目,旨在为Web应用系统提供一种可靠的单点登录解决方法(属于Web SSO),可以理解SSO是协议,CAS是SSO的实现之一.  
### (3). CAS主要特性
+ 开源的、多协议的 SSO 解决方案;Protocols: Custom Protocol 、 CAS 、 OAuth 、 OpenID 、 RESTful API 、 SAML1.1 、 SAML2.0等.   
+ 支持多种认证机制: Active Directory 、 JAAS 、 JDBC 、 LDAP 、 X.509 Certificates等.   
+ 安全策略: 使用票据(Ticket)来实现支持的认证协议.  
+ 支持授权: 可以决定哪些服务可以请求和验证服务票据(Service Ticket).   
+ 提供高可用性: 通过把认证过的状态数据存储在TicketRegistry 组件中,这些组件有很多支持分布式环境的实现,如: BerkleyDB、Default 、EhcacheTicketRegistry 、JDBCTicketRegistry 、JBOSS TreeCache 、 JpaTicketRegistry 、 MemcacheTicketRegistry等.  
### (4). 

### (5). 

### (6). 

### (7). 

### (8). 

### (9). 

### (10). 

