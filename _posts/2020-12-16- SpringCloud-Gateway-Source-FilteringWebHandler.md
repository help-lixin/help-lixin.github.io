---
layout: post
title: 'Spring Cloud Gateway FilteringWebHandler(七)'
date: 2020-12-16
author: 李新
tags: SpringCloudGateway源码
---

### (1). 概述
> 在前面(第六节),分析了:RoutePredicateHandlerMapping,它主要获取用户配置:Route中的断言(Predicate)对象.调用apply(Exchange).获取符合断言的Route对象.并把这个Route设置到:Exchange中.所以:Gateway是首先执行断言(Predicate).      
> 这一节,主要讲解:Filter的执行.    

### (2). FilteringWebHandler类图
> 1. 所有的:GlobalFilter都会进行适配成:GatewayFilterAdapter(典型的适配模式),而GatewayFilterAdapter,又会被:OrderedGatewayFilter对象包裹.   

!["FilteringWebHandler"](/assets/spring-cloud-gateway/imgs/spring-cloud-gateway-filtering-web-handler-class.jpg)

### (2). FilteringWebHandler
```
public class FilteringWebHandler 
	    // ***********************************************
       // SimpleHandlerAdapter会回调handler方法.
	   // ***********************************************
       implements WebHandler {

    private final List<GatewayFilter> globalFilters;

	public FilteringWebHandler(List<GlobalFilter> globalFilters) {
		// 2.1 对globalFilters进行加工处理
		this.globalFilters = loadFilters(globalFilters);
	}

	private static List<GatewayFilter> loadFilters(List<GlobalFilter> filters) {
		// 2.2 适配并加工
		return filters.stream()
				.map(filter -> {
					// **********************************************
					// 适配模式,把GlobalFilter适配成:GatewayFilter
					// **********************************************
					GatewayFilterAdapter gatewayFilter = new GatewayFilterAdapter(filter);

					// 判断是否为:Ordered排序的子类
					if (filter instanceof Ordered) {
						int order = ((Ordered) filter).getOrder();
						// Wrapper模式
						return new OrderedGatewayFilter(gatewayFilter, order);
					}
					return gatewayFilter;
				}).collect(Collectors.toList());
	} // end loadFilters


	// reactor-netty解析后,最终会调用:handler.
	public Mono<Void> handle(ServerWebExchange exchange) {
		// 获取用户级别的:GatewayFilter
		// GATEWAY_ROUTE_ATTR是在:
		// RoutePredicateHandlerMapping.getHandlerInternal
		// 进行设置的.
		Route route = exchange.getRequiredAttribute(GATEWAY_ROUTE_ATTR);
		List<GatewayFilter> gatewayFilters = route.getFilters();

		// 获取全局级别的:GatewayFilter(GlobalFilter已经被适配成:GatewayFilter)
		List<GatewayFilter> combined = new ArrayList<>(this.globalFilters);
		combined.addAll(gatewayFilters);
		//TODO: needed or cached?
		// 排序
		AnnotationAwareOrderComparator.sort(combined);

		if (logger.isDebugEnabled()) {
			logger.debug("Sorted gatewayFilterFactories: "+ combined);
		}

		// 挨个调用相应的:filter方法
		return new DefaultGatewayFilterChain(combined).filter(exchange);
	}// end handle

}
```
### (3). 总结
> 1. FilteringWebHandler是WebHandler的实现类,它负责把GlobalFilter转换成:GatewayFilter.      
> 2. handler在执行时,获得配置的:GatewayFilter和全局GatewayFilterAdapter.调用相应的:filter方法.     

