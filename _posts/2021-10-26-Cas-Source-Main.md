---
layout: post
title: 'CAS分析源码之切入点在哪(四)' 
date: 2021-10-26
author: 李新
tags:  CAS
---

### (1). 概述
在前面,把CAS的源码拉取下来了,并编译通过,也对目录结构进行了分析,那么Cas的切入点到底在哪呢?              

### (2). 切入点在哪?
> 我想偷懒,所以,用一种简单的方式.把断点打在:javax.servlet.FilterChain的实现类(org.apache.catalina.core.ApplicationFilterChain)里,查看有多少自定义的Filter托管给了Servlet容器.   

### (3). 托管给Web容器的相关Filter
```
# 我们只剖析跟名称:org.apereo相关的Filter,其余Filter不分析.
ApplicationFilterConfig[name=characterEncodingFilter, filterClass=org.springframework.boot.web.servlet.filter.OrderedCharacterEncodingFilter]

ApplicationFilterConfig[name=CAS Client Info Logging Filter, filterClass=org.apereo.inspektr.common.web.ClientInfoThreadLocalFilter]
ApplicationFilterConfig[name=threadContextMDCServletFilter, filterClass=org.apereo.cas.logging.web.ThreadContextMDCServletFilter]

// ************************************************************************************************************
// 以下这四个Filter是Spring boot的.
// ************************************************************************************************************
ApplicationFilterConfig[name=webMvcMetricsFilter, filterClass=org.springframework.boot.actuate.metrics.web.servlet.WebMvcMetricsFilter]
ApplicationFilterConfig[name=formContentFilter, filterClass=org.springframework.boot.web.servlet.filter.OrderedFormContentFilter]
ApplicationFilterConfig[name=requestContextFilter, filterClass=org.springframework.boot.web.servlet.filter.OrderedRequestContextFilter]
ApplicationFilterConfig[name=springSecurityFilterChain, filterClass=org.springframework.boot.web.servlet.DelegatingFilterProxyRegistrationBean$1]

ApplicationFilterConfig[name=responseHeadersFilter, filterClass=org.apereo.cas.web.support.filters.AddResponseHeadersFilter]
ApplicationFilterConfig[name=responseHeadersSecurityFilter, filterClass=org.apereo.cas.services.web.support.RegisteredServiceResponseHeadersEnforcementFilter]
ApplicationFilterConfig[name=requestParameterSecurityFilter, filterClass=org.apereo.cas.web.support.filters.RequestParameterPolicyEnforcementFilter]
ApplicationFilterConfig[name=currentCredentialsAndAuthenticationClearingFilter, filterClass=org.apereo.cas.web.support.AuthenticationCredentialsThreadLocalBinderClearingFilter]
```
### (4). 总结
通过上面的剖析方式,知道以下这些Filter属于Cas的扩展点(后面,将会一个一个的分析这些Filter):   

+ ClientInfoThreadLocalFilter
+ ThreadContextMDCServletFilter
+ DelegatingFilterProxyRegistrationBean
+ AddResponseHeadersFilter
+ RegisteredServiceResponseHeadersEnforcementFilter
+ RequestParameterPolicyEnforcementFilter
+ AuthenticationCredentialsThreadLocalBinderClearingFilter