---
layout: post
title: 'CAS源码编译并运行(二)' 
date: 2021-10-26
author: 李新
tags:  CAS
---

### (1). 概述
在这里我使用cas-overlay-template来搭建Cas.  

### (2). 环境要求
+ JDK11
### (3). 源码构建
```
# 1. 进入工作目录
lixin@lixin ~ % cd ~/GitWorkspace
# 2. 下载源码
lixin@lixin GitWorkspace % git clone https://github.com/help-lixin/cas-overlay-template.git
# 3. 进入cas-overlay-template目录
lixin@lixin GitWorkspace % cd cas-overlay-template
# 4. 构建源码(构建过程会比较漫长)
lixin@lixin cas-overlay-template % ./gradlew clean build -x test
```
### (4). 配置登录的账号和密码
```
infinova@lixin cas-overlay-template % more  src/main/resources/application.yml
# Application properties that need to be
# embedded within the web application can be included here
cas:
  authn:
    accept:
      users: lixin::123456
```
### (5). 创建配置
```
# 拷贝源码目录下的配置到指定配置目录
infinova@lixin cas-overlay-template % sudo cp -rf etc/cas /etc
# 修改目录权限
lixin@lixin cas-overlay-template % sudo chown -R lixin:staff /etc/cas
```
### (6). 自签证书
```
#  自签证书
infinova@lixin cas-overlay-template % ./gradlew createKeystore
```
### (7). 运行构建后的应用
```
lixin@lixin cas-overlay-template % java -Xdebug -Xrunjdwp:transport=dt_socket,address=5000,server=y,suspend=y -jar ./build/libs/cas.war
```
### (8). WEB测试
!["cas 登录"](/assets/cas/imgs/cas-login.png)  
!["cas 首页"](/assets/cas/imgs/cas-home.png)   

### (9). 查看控制台信息
```
2021-10-29 10:09:09,201 INFO [org.apereo.cas.web.CasWebApplication] - <>
2021-10-29 10:09:09,201 INFO [org.apereo.cas.web.CasWebApplication] - <Ready to process requests @ [2021-10-29T02:09:09.200Z]>
2021-10-29 10:09:09,218 INFO [org.apereo.cas.services.AbstractServicesManager] - <Loaded [0] service(s) from [InMemoryServiceRegistry].>
2021-10-29 10:09:39,216 INFO [org.apereo.cas.ticket.registry.DefaultTicketRegistryCleaner] - <[0] expired tickets removed.>
2021-10-29 10:11:14,000 WARN [javax.persistence.spi] - <javax.persistence.spi::No valid providers found.>
2021-10-29 10:11:14,086 INFO [org.apereo.inspektr.audit.support.Slf4jLoggingAuditTrailManager] - <Audit trail record BEGIN
=============================================================
WHO: audit:unknown
WHAT: {source=RankedMultifactorAuthenticationProviderWebflowEventResolver, event=success, timestamp=Fri Oct 29 10:11:14 CST 2021}
ACTION: AUTHENTICATION_EVENT_TRIGGERED
APPLICATION: CAS
WHEN: Fri Oct 29 10:11:14 CST 2021
CLIENT IP ADDRESS: 0:0:0:0:0:0:0:1
SERVER IP ADDRESS: 0:0:0:0:0:0:0:1
=============================================================

>
2021-10-29 10:11:27,134 INFO [org.apereo.cas.authentication.DefaultAuthenticationManager] - <Authenticated principal [lixin] with attributes [{}] via credentials [[UsernamePasswordCredential(username=lixin, source=null, customFields={})]].>
2021-10-29 10:11:27,135 INFO [org.apereo.inspektr.audit.support.Slf4jLoggingAuditTrailManager] - <Audit trail record BEGIN
=============================================================
WHO: lixin
WHAT: [UsernamePasswordCredential(username=lixin, source=null, customFields={})]
ACTION: AUTHENTICATION_SUCCESS
APPLICATION: CAS
WHEN: Fri Oct 29 10:11:27 CST 2021
CLIENT IP ADDRESS: 0:0:0:0:0:0:0:1
SERVER IP ADDRESS: 0:0:0:0:0:0:0:1
=============================================================

>
2021-10-29 10:11:27,167 INFO [org.apereo.inspektr.audit.support.Slf4jLoggingAuditTrailManager] - <Audit trail record BEGIN
=============================================================
WHO: lixin
WHAT: TGT-1-*****Pu2S-oN6zE-lixin
ACTION: TICKET_GRANTING_TICKET_CREATED
APPLICATION: CAS
WHEN: Fri Oct 29 10:11:27 CST 2021
CLIENT IP ADDRESS: 0:0:0:0:0:0:0:1
SERVER IP ADDRESS: 0:0:0:0:0:0:0:1
=============================================================

>
2021-10-29 10:11:39,219 INFO [org.apereo.cas.ticket.registry.DefaultTicketRegistryCleaner] - <[0] expired tickets removed.>
2021-10-29 10:11:45,298 INFO [org.apereo.cas.logout.DefaultLogoutManager] - <Performing logout operations for [TGT-1-*****Pu2S-oN6zE-lixin]>
2021-10-29 10:11:45,301 INFO [org.apereo.cas.logout.DefaultLogoutManager] - <[0] logout requests were processed>
2021-10-29 10:11:45,302 INFO [org.apereo.inspektr.audit.support.Slf4jLoggingAuditTrailManager] - <Audit trail record BEGIN
=============================================================
WHO: lixin
WHAT: TGT-1-*****Pu2S-oN6zE-lixin
ACTION: TICKET_DESTROYED
APPLICATION: CAS
WHEN: Fri Oct 29 10:11:45 CST 2021
CLIENT IP ADDRESS: 0:0:0:0:0:0:0:1
SERVER IP ADDRESS: 0:0:0:0:0:0:0:1
=============================================================
```