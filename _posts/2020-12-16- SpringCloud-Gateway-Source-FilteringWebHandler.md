---
layout: post
title: 'Spring Cloud Gateway FilteringWebHandler Web入口(七)'
date: 2020-12-16
author: 李新
tags: SpringCloudGateway源码
---

### (1). FilteringWebHandler类图
> 1. 从类图上分析,可得出结论:FilteringWebHandler是WebHandler(WebFlux的入口)的实现类.   
> 2. 所有的:GlobalFilter都会进行适配成:GatewayFilterAdapter(典型的适配模式),而GatewayFilterAdapter,又会被:OrderedGatewayFilter对象包裹.   
> 3. 从图中能看到:<font color='red'>所有的GlobalFilter实现,其中:ForwardRoutingFilter会调用:HandlerMapping(WebFlux中URL与Handler的映射).</font>    

!["FilteringWebHandler"](/assets/spring-cloud-gateway/imgs/spring-cloud-gateway-filtering-web-handler-class.jpg)

### (2). 
```
public class FilteringWebHandler 
	    // ***********************************************
       // WebHandler是WebFlux的入口程序,相当于:DispatcherServlet一样
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
> 1. FilteringWebHandler是WebHandler的实现类,是Web程序的入口.它负责把GlobalFilter转换成:GatewayFilter.      
> 2. handler在执行时,获得配置的:GatewayFilter和全局GatewayFilterAdapter.调用相应的:filter方法.     
> 3. <font color='red'>ForwardRoutingFilter是最后一个filter,它最终会调用:HandlerMapping.</font>    

