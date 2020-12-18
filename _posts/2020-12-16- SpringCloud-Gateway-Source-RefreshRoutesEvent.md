---
layout: post
title: 'Spring Cloud Gateway RouteLocator之RefreshRoutesEvent(八)'
date: 2020-12-16
author: 李新
tags: SpringCloudGateway源码
---

### (1). 概述
>  GatewayProperties是Gateway的配置载体,那么,配置能否动态呢? 

### (2). 查看RouteLocator的实现类:
> 1. <font color='red'>此处的刷新,已经是被RouteDefinitionRouteLocator解析后的:Route,而不是:RouteDefinition,RouteDefinition是元数据,Route是元数据解析之后的对象.</font>     
> 2. RouteDefinitionRouteLocator 我们在前面的章节有分析过了,在此不再讲述.    
> 3. CachingRouteLocator 这是一个,带有缓存功能的.还实现了:ApplicationListener<RefreshRoutesEvent>.从实现的事件来判断,应该是支持自动刷新的.    
> 4. CompositeRouteLocator 一看就知道是一个组合模式了.      
> 5. 查看RouteLocator的构建过程.以证实.    

```
// 1. 构建:RouteLocator
@Bean
public RouteLocator routeDefinitionRouteLocator(
		GatewayProperties properties,
		List<GatewayFilterFactory> GatewayFilters,
		List<RoutePredicateFactory> predicates,
		RouteDefinitionLocator routeDefinitionLocator,
		@Qualifier("webFluxConversionService")
		ConversionService conversionService) {

	return new RouteDefinitionRouteLocator(routeDefinitionLocator, predicates, GatewayFilters,
				properties, conversionService);
}// end routeDefinitionRouteLocator

// 2. 构建:CompositeRouteLocator和CachingRouteLocator并包裹着:RouteDefinitionRouteLocator
// 并标注为:@Primary,自然,注入:RoutePredicateHandlerMapping里的就是:CachingRouteLocator
@Bean
@Primary
public RouteLocator cachedCompositeRouteLocator(List<RouteLocator> routeLocators) {
	return new CachingRouteLocator(
		   new CompositeRouteLocator(Flux.fromIterable(routeLocators))
		   );
} // end cachedCompositeRouteLocator
```

!["RouteLocator接口的实现类"](/assets/spring-cloud-gateway/imgs/spring-cloud-gateway-route-locator.jpg)

### (3). CachingRouteLocator
```
public class CachingRouteLocator implements 
             RouteLocator, 
			 // 实现了Spring的:ApplicationListener
			 ApplicationListener<RefreshRoutesEvent> {
    
    private final RouteLocator delegate;
	private final Flux<Route> routes;
	private final Map<String, List> cache = new HashMap<>();

	public CachingRouteLocator(RouteLocator delegate) {
		this.delegate = delegate;
		// routes是cache,当没有内容时,会触发:delegate去请求
		// 此处delegate = CompositeRouteLocator
		routes = CacheFlux.lookup(cache, "routes", Route.class)
				.onCacheMissResume(() -> this.delegate.getRoutes().sort(AnnotationAwareOrderComparator.INSTANCE));
	}// end 构造器

	@Override
	public Flux<Route> getRoutes() {
		return this.routes;
	}

	
	public Flux<Route> refresh() {
		// 清除缓存
		this.cache.clear();
		return this.routes;
	}

	@Override
	public void onApplicationEvent(RefreshRoutesEvent event) {
		// 实现事件监听
		refresh();
	} 
}
```
### (4). 总结
> 改变配置后,只需要触发:RefreshRoutesEvent事件,就可以触发重新加载Route对象了.   
> <font color='red'>加载的是:Route,那么:RouteDefinition可否自动重新加载呢?要看下一节的剖析</font>   
