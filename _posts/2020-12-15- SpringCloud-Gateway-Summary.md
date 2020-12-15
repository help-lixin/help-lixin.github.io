---
layout: post
title: 'SpringCloud Gateway 概念(一)'
date: 2020-12-15
author: 李新
tags: SpringCloudGateway
---

### (1). Spring Cloud Gatewary概念
> Spring Cloud Gatewary以以下三大部份组成: Route / Predicate / Filter.

### (2). Route
> 路由是网关最基础的部份,它由:Route ID,Target URI,Collection<Predicate>三部份组成.   
> Spring Cloud Gateway包含许多内置的:Route Predicate Factoies. 
> RoutePredicateFactory是Route对象的工厂,其实现包括如下:

!["RoutePredicateFactory"](/assets/spring-cloud-gateway/imgs/spring-cloud-gateway-route-predicate-factory.jpg)


### (3). Predicate
> Predicate是JDK8的断言函数,它的输入类型为:org.springframework.web.server.ServerWebExchange.它允许开发者去定义匹配来自于Http Reqeust中的任何信息,比如:Header或参数等.    

### (4). Filter
> Filter是一个标准的过滤器,Spring Cloud Gateway中的Filter分两种类型,分别是:Gateway Filter和Gloabl Filter,过滤器可对request和response进行处理.   
> Gateway Filter : 网关过滤器,需要通过spring.cloud.routes.filters配置在具体的路由下,只能作用在当前的路由上或者通过spring.cloud.default-filters配置在全局,作用在所有的路由上.   
> Global Filter: 全局过滤器,不需要在配置文件中配置,作用在所有的路由上,最终通过GatewayFilterAdapter包装成GatewayFilterChain可识的的过滤器,它为请求业务以及路由的URI转换为真实业务服务请求地址的核心过滤器,不需要配置系统初始化时加载,并作用在每个路由上.   

> GatewayFilterFactory的实现子类如下:

!["GatewayFilterFactory实现子类"](/assets/spring-cloud-gateway/imgs/spring-cloud-gateway-gateway-filter-factory.jpg)

> GlobalFilter的实现子类如下:

!["GlobalFilter的实现子类"](/assets/spring-cloud-gateway/imgs/spring-cloud-gateway-global-filter.jpg)


### (5). Spring Cloud Gateway原理图

!["Spring Cloud Gateway原理"](/assets/spring-cloud-gateway/imgs/spring_cloud_gateway_diagram.png)
