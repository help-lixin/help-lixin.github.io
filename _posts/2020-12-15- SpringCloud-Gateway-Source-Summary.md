---
layout: post
title: 'Spring Cloud Gateway 源码总结'
date: 2020-12-15
author: 李新
tags: SpringCloudGateway源码
---

### (1). Spring Cloud Gateway源码索引目录
> 1. Spring Cloud Gateway业务模型: ["GatewayProperties"](https://www.lixin.help/2020/12/16/SpringCloud-Gateway-Source-GatewayProperties.html)
> 2. Filter: ["GatewayFilterFactory"](https://www.lixin.help/2020/12/16/SpringCloud-Gateway-Source-GatewayFilterFactory.html)
> 3. Predicate: ["RoutePredicateFactory"](https://www.lixin.help/2020/12/16/SpringCloud-Gateway-Source-RoutePredicateFactory.html)
> 4. 配置(Configuration)文件详解: ["GatewayAutoConfiguration"](https://www.lixin.help/2020/12/16/SpringCloud-Gateway-Source-GatewayAutoConfiguration.html)
> 5. 加载(RouteDefinition)转换成:Route对象: ["RouteDefinitionRouteLocator"](https://www.lixin.help/2020/12/16/SpringCloud-Gateway-Source-RouteLocator.html)
> 6. HandlerMapping: ["RoutePredicateHandlerMapping"](https://www.lixin.help/2020/12/16/SpringCloud-Gateway-Source-RoutePredicateHandlerMapping.html)
> 7. WebHandler: ["FilteringWebHandler"](https://www.lixin.help/2020/12/16/SpringCloud-Gateway-Source-FilteringWebHandler.html)
> 8. Route动态更新: ["RouteLocator-RefreshRoutesEvent"](https://www.lixin.help/2020/12/16/SpringCloud-Gateway-Source-RefreshRoutesEvent.html)
> 9. RouteDefinition动态更新: ["RouteDefinitionLocator"](https://www.lixin.help/2020/12/16/SpringCloud-Gateway-Source-RouteDefinitionLocator.html)