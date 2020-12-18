---
layout: post
title: 'Spring Cloud Gateway RoutePredicateHandlerMapping(六)'
date: 2020-12-16
author: 李新
tags: SpringCloudGateway源码 
---

### (1). WebFlux DispatcherHandler
> DispatcherHandler的调用过程请参考下面这个链接:   
["DispatcherHandler的调用过程"](https://www.lixin.help/2020/12/18/Spring-WebFlux-DispatcherHandler.html)

### (2). RoutePredicateHandlerMapping
```
public class RoutePredicateHandlerMapping extends 
             AbstractHandlerMapping {

    private final FilteringWebHandler webHandler;
	private final RouteLocator routeLocator;
	private final Integer managmentPort;

	public RoutePredicateHandlerMapping(
            // *************************************************************
            // RoutePredicateHandlerMapping构造器依赖于:WebHandler(FilteringWebHandler)
            // *************************************************************
            FilteringWebHandler webHandler, 
            RouteLocator routeLocator, 
            GlobalCorsProperties globalCorsProperties, 
            Environment environment) {
		this.webHandler = webHandler;
		this.routeLocator = routeLocator;

		if (environment.containsProperty("management.server.port")) {
			managmentPort = new Integer(environment.getProperty("management.server.port"));
		} else {
			managmentPort = null;
		}
		setOrder(1);		
		setCorsConfigurations(globalCorsProperties.getCorsConfigurations());
	}// end 构造器

    protected Mono<?> getHandlerInternal(ServerWebExchange exchange) {
		// don't handle requests on the management port if set
        // 如果请求的是管理端口,返回:empty
		if (managmentPort != null && exchange.getRequest().getURI().getPort() == managmentPort.intValue()) {
			return Mono.empty();
		}

		exchange.getAttributes().put(GATEWAY_HANDLER_MAPPER_ATTR, getSimpleName());

        // ******************************************************************
        // 3. 重点:RoutePredicateHandlerMapping.lookupRoute
        // ******************************************************************
		return lookupRoute(exchange)
				// .log("route-predicate-handler-mapping", Level.FINER) //name this
				.flatMap((Function<Route, Mono<?>>) r -> {

					exchange.getAttributes().remove(GATEWAY_PREDICATE_ROUTE_ATTR);
					if (logger.isDebugEnabled()) {
						logger.debug("Mapping [" + getExchangeDesc(exchange) + "] to " + r);
					}

                    // *************************************************
                    // 这是重点:
                    // exchange = DefaultServerWebExchange
                    // Predicate.apply(DefaultServerWebExchange)之后,会返回一个:Route,之后把这个Route对象设置到:
                    // DefaultServerWebExchange.attributes属性中.因为,后面会要从:DefaultServerWebExchange中再次获取出来用.
                    // 实际就是为了解藕.
                    // 从DefaultServerWebExchange为每一交请求,对应一个:DefaultServerWebExchange,所以是安全的.
                    // *************************************************
					exchange.getAttributes().put(GATEWAY_ROUTE_ATTR, r);

                    // *************************************************
                    // 返回的是:WebHandler(FilteringWebHandler)
                    // *************************************************
					return Mono.just(webHandler);
				}).switchIfEmpty(Mono.empty().then(Mono.fromRunnable(() -> {
					exchange.getAttributes().remove(GATEWAY_PREDICATE_ROUTE_ATTR);
					if (logger.isTraceEnabled()) {
						logger.trace("No RouteDefinition found for [" + getExchangeDesc(exchange) + "]");
					}
				})));
	} // end getHandlerInternal
}
```
### (3). RoutePredicateHandlerMapping.lookupRoute
```
protected Mono<Route> lookupRoute(ServerWebExchange exchange) {
    // 这部份的业务逻是:
    // 1. 遍历(List)Route
    // 2. 调用:Predicate(比如:PathRoutePredicateFactory),返回符合条件的:Route.
    // 3. 验证:Route对象的参数.
    // 此处的:routeLocator实际就是:spring.cloud.gateway.routes
    return this.routeLocator
        .getRoutes()
        // start concatMap
        .concatMap(route -> 
                Mono
                .just(route)
                .filterWhen(r -> {   
                    // filterWhen为过滤符合条件的Route对象
                    // add the current route we are testing
                    exchange.getAttributes().put(GATEWAY_PREDICATE_ROUTE_ATTR, r.getId());

                    // **********************************************************
                    // r.getPredicate() = 
                    // [PathRoutePredicateFactory.Config patterns = list['/api-gateway/**'], matchOptionalTrailingSeparator = true]
                    // **********************************************************
                    return r.getPredicate().apply(exchange);
                })
                //instead of immediately stopping main flux due to error, log and swallow it
                .doOnError(e -> logger.error("Error applying predicate for route: "+route.getId(), e))
                .onErrorResume(e -> Mono.empty())
        )
        // end concatMap
        .next() // 只取一个
        .map(route -> {
            if (logger.isDebugEnabled()) {
                logger.debug("Route matched: " + route.getId());
            }
            // 验证route
            validateRoute(route, exchange);
            return route;
        });
}// end lookupRoute
```
### (4). 总结
> 1. DispatcherHandler接受所有的请求.       
> 2. 遍历所有([RouterFunctionMapping, RequestMappingHandlerMapping, RoutePredicateHandlerMapping,SimpleUrlHandlerMapping])的HandlerMapping.此处:RoutePredicateHandlerMapping符合规则.         
> 3. 调用RoutePredicateHandlerMapping.getHandler方法.       
> 4. getHandler方法,内部首先:调用RoutePredicateHandlerMapping.lookupRoute方法(实则就是遍历用户配置的:routes),并调用用户配置的:Predicate.用以判断URL是否符合条件,如果符合:返回符合的:Route对象.      
> 5. getHandler方法,内部再次:调用flatMap方法,返回:Mono<FilteringWebHandler>         
> 6. <font color='red'>所以,Gateway是先调用:Predicate.</font>    
> 7. 最后,就是Filter的执行了.Filter的执行在:FilteringWebHandler的handler方法里.这部份内容,将另开一节再剖析.    
