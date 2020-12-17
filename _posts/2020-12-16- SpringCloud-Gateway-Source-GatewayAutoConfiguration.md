---
layout: post
title: 'Spring Cloud Gateway GatewayAutoConfiguration(四)'
date: 2020-12-16
author: 李新
tags: SpringCloudGateway源码
---

### (1). GatewayAutoConfiguration
> GatewayAutoConfiguration固名思意,就是针对:Gateway进行配置的类.

### (2). 注册:GatewayFilterFactory
> 向Spring容器中注册大量的:GatewayFilterFactory的实现子类.  

```
@Bean
public AddRequestHeaderGatewayFilterFactory addRequestHeaderGatewayFilterFactory() {
	return new AddRequestHeaderGatewayFilterFactory();
}

@Bean
public AddRequestParameterGatewayFilterFactory addRequestParameterGatewayFilterFactory() {
	return new AddRequestParameterGatewayFilterFactory();
}

@Bean
public AddResponseHeaderGatewayFilterFactory addResponseHeaderGatewayFilterFactory() {
	return new AddResponseHeaderGatewayFilterFactory();
}

// Hystrix存在的情况下
@Bean
public HystrixGatewayFilterFactory hystrixGatewayFilterFactory(ObjectProvider<DispatcherHandler> dispatcherHandler) {
	return new HystrixGatewayFilterFactory(dispatcherHandler);
}

@Bean
public FallbackHeadersGatewayFilterFactory fallbackHeadersGatewayFilterFactory() {
	return new FallbackHeadersGatewayFilterFactory();
}

@Bean
public ModifyRequestBodyGatewayFilterFactory modifyRequestBodyGatewayFilterFactory(ServerCodecConfigurer codecConfigurer) {
	return new ModifyRequestBodyGatewayFilterFactory(codecConfigurer);
}

@Bean
public ModifyResponseBodyGatewayFilterFactory modifyResponseBodyGatewayFilterFactory(ServerCodecConfigurer codecConfigurer) {
	return new ModifyResponseBodyGatewayFilterFactory(codecConfigurer);
}

@Bean
public PrefixPathGatewayFilterFactory prefixPathGatewayFilterFactory() {
	return new PrefixPathGatewayFilterFactory();
}

@Bean
public PreserveHostHeaderGatewayFilterFactory preserveHostHeaderGatewayFilterFactory() {
	return new PreserveHostHeaderGatewayFilterFactory();
}

@Bean
public RedirectToGatewayFilterFactory redirectToGatewayFilterFactory() {
	return new RedirectToGatewayFilterFactory();
}

@Bean
public RemoveRequestHeaderGatewayFilterFactory removeRequestHeaderGatewayFilterFactory() {
	return new RemoveRequestHeaderGatewayFilterFactory();
}

@Bean
public RemoveResponseHeaderGatewayFilterFactory removeResponseHeaderGatewayFilterFactory() {
	return new RemoveResponseHeaderGatewayFilterFactory();
}

@Bean
@ConditionalOnBean({RateLimiter.class, KeyResolver.class})
public RequestRateLimiterGatewayFilterFactory requestRateLimiterGatewayFilterFactory(RateLimiter rateLimiter, KeyResolver resolver) {
	return new RequestRateLimiterGatewayFilterFactory(rateLimiter, resolver);
}

@Bean
public RewritePathGatewayFilterFactory rewritePathGatewayFilterFactory() {
	return new RewritePathGatewayFilterFactory();
}

@Bean
public RetryGatewayFilterFactory retryGatewayFilterFactory() {
	return new RetryGatewayFilterFactory();
}

@Bean
public SetPathGatewayFilterFactory setPathGatewayFilterFactory() {
	return new SetPathGatewayFilterFactory();
}

@Bean
public SecureHeadersGatewayFilterFactory secureHeadersGatewayFilterFactory(SecureHeadersProperties properties) {
	return new SecureHeadersGatewayFilterFactory(properties);
}

@Bean
public SetRequestHeaderGatewayFilterFactory setRequestHeaderGatewayFilterFactory() {
	return new SetRequestHeaderGatewayFilterFactory();
}

@Bean
public SetResponseHeaderGatewayFilterFactory setResponseHeaderGatewayFilterFactory() {
	return new SetResponseHeaderGatewayFilterFactory();
}

@Bean
public RewriteResponseHeaderGatewayFilterFactory rewriteResponseHeaderGatewayFilterFactory() {
	return new RewriteResponseHeaderGatewayFilterFactory();
}

@Bean
public SetStatusGatewayFilterFactory setStatusGatewayFilterFactory() {
	return new SetStatusGatewayFilterFactory();
}

@Bean
public SaveSessionGatewayFilterFactory saveSessionGatewayFilterFactory() {
	return new SaveSessionGatewayFilterFactory();
}

@Bean
public StripPrefixGatewayFilterFactory stripPrefixGatewayFilterFactory() {
	return new StripPrefixGatewayFilterFactory();
}

@Bean
public RequestHeaderToRequestUriGatewayFilterFactory requestHeaderToRequestUriGatewayFilterFactory() {
	return new RequestHeaderToRequestUriGatewayFilterFactory();
}

@Bean
public RequestSizeGatewayFilterFactory requestSizeGatewayFilterFactory() {
	return new RequestSizeGatewayFilterFactory();
}

```
### (3). 注册:RoutePredicateFactory
>  向Spring容器中,注册:RoutePredicateFactory的实现类. 

```
@Bean
public AfterRoutePredicateFactory afterRoutePredicateFactory() {
	return new AfterRoutePredicateFactory();
}

@Bean
public BeforeRoutePredicateFactory beforeRoutePredicateFactory() {
	return new BeforeRoutePredicateFactory();
}

@Bean
public BetweenRoutePredicateFactory betweenRoutePredicateFactory() {
	return new BetweenRoutePredicateFactory();
}

@Bean
public CookieRoutePredicateFactory cookieRoutePredicateFactory() {
	return new CookieRoutePredicateFactory();
}

@Bean
public HeaderRoutePredicateFactory headerRoutePredicateFactory() {
	return new HeaderRoutePredicateFactory();
}

@Bean
public HostRoutePredicateFactory hostRoutePredicateFactory() {
	return new HostRoutePredicateFactory();
}

@Bean
public MethodRoutePredicateFactory methodRoutePredicateFactory() {
	return new MethodRoutePredicateFactory();
}

@Bean
public PathRoutePredicateFactory pathRoutePredicateFactory() {
	return new PathRoutePredicateFactory();
}

@Bean
public QueryRoutePredicateFactory queryRoutePredicateFactory() {
	return new QueryRoutePredicateFactory();
}

@Bean
public ReadBodyPredicateFactory readBodyPredicateFactory() {
	return new ReadBodyPredicateFactory();
}

@Bean
public RemoteAddrRoutePredicateFactory remoteAddrRoutePredicateFactory() {
	return new RemoteAddrRoutePredicateFactory();
}

@Bean
@DependsOn("weightCalculatorWebFilter")
public WeightRoutePredicateFactory weightRoutePredicateFactory() {
	return new WeightRoutePredicateFactory();
}

@Bean
public CloudFoundryRouteServiceRoutePredicateFactory cloudFoundryRouteServiceRoutePredicateFactory() {
	return new CloudFoundryRouteServiceRoutePredicateFactory();
}
```
### (4). 注册:GatewayProperties
>  <font color='red'>GatewayProperties保存在application.yml配置的路由信息.</font>

```
@Bean
public GatewayProperties gatewayProperties() {
	return new GatewayProperties();
}

// *************************************************************
// 通过:PropertiesRouteDefinitionLocator包裹着route配置信息.
// *************************************************************
@Bean
@ConditionalOnMissingBean
public PropertiesRouteDefinitionLocator propertiesRouteDefinitionLocator(GatewayProperties properties) {
	return new PropertiesRouteDefinitionLocator(properties);
} // end

// **********************************************************
// 创建:routeDefinitionLocator
// **********************************************************
// 组合多个:RouteDefinitionLocator(PropertiesRouteDefinitionLocator/InMemoryRouteDefinitionRepository)
@Bean
@Primary
public RouteDefinitionLocator routeDefinitionLocator(
	   List<RouteDefinitionLocator> routeDefinitionLocators) {

	return new CompositeRouteDefinitionLocator(Flux.fromIterable(routeDefinitionLocators));
} // end 
```

### (5). 注册:GlobalFilter
```

@Bean
public AdaptCachedBodyGlobalFilter adaptCachedBodyGlobalFilter() {
	return new AdaptCachedBodyGlobalFilter();
}

@Bean
public RouteToRequestUrlFilter routeToRequestUrlFilter() {
	return new RouteToRequestUrlFilter();
}

@Bean
public ForwardRoutingFilter forwardRoutingFilter(ObjectProvider<DispatcherHandler> dispatcherHandler) {
	return new ForwardRoutingFilter(dispatcherHandler);
}

@Bean
public ForwardPathFilter forwardPathFilter() {
	return new ForwardPathFilter();
}

// ******************************************************
// 通过:FilteringWebHandler,包裹所有的:GlobalFilter.
// FilteringWebHandler是WebHandler(WebFlux)的实现子类
// ******************************************************
@Bean
public FilteringWebHandler filteringWebHandler(List<GlobalFilter> globalFilters) {
	return new FilteringWebHandler(globalFilters);
} // end filteringWebHandler
```

### (6). 创建:RouteLocator
```
@Bean
public RouteLocator routeDefinitionRouteLocator(
	                   GatewayProperties properties,
					   List<GatewayFilterFactory> GatewayFilters,
					   List<RoutePredicateFactory> predicates,
					   RouteDefinitionLocator routeDefinitionLocator,
					   @Qualifier("webFluxConversionService")
					   ConversionService conversionService) {
    
    // 创建:RouteDefinitionRouteLocator
	return new RouteDefinitionRouteLocator(
				routeDefinitionLocator, 
				predicates, 
				GatewayFilters,
				properties, 
				conversionService);
}// end RouteLocator


// 对RouteLocator进行缓存.
@Bean
@Primary
public RouteLocator cachedCompositeRouteLocator(List<RouteLocator> routeLocators) {
	return new CachingRouteLocator(new CompositeRouteLocator(Flux.fromIterable(routeLocators)));
} // end

// ******************************************************************************
// RoutePredicateHandlerMapping是核心:
// 它包含: 
//    RouteLocator [GatewayFilterFactory/RoutePredicateFactory/RouteDefinitionLocator]
//    FilteringWebHandler [  AdaptCachedBodyGlobalFilter/RouteToRequestUrlFilter/ForwardRoutingFilter/ForwardPathFilter/... ]
// ******************************************************************************
@Bean
public RoutePredicateHandlerMapping routePredicateHandlerMapping(
		FilteringWebHandler webHandler, 
		RouteLocator routeLocator,
		GlobalCorsProperties globalCorsProperties, 
		Environment environment) {
    // 
	return new RoutePredicateHandlerMapping(
		                  webHandler, 
						  routeLocator,
						  globalCorsProperties, 
						  environment);
}

```

### (7). 总结
> GatewayAutoConfiguration负责:    
> 1. 创建多个:GatewayFilterFactory.   
> 2. 创建多个:RoutePredicateFactory.   
> 3. 把GatewayFilterFactory和RoutePredicateFactory组合成:RouteDefinitionRouteLocator.   
> 4. 创建:FilteringWebHandler内部聚合多个:GlobalFilter.<font color='red'>FilteringWebHandler是WebHandler(WebFlux)的子类,与:DispatcherHandler是平级的.</font>    
> 5. 创建:RoutePredicateHandlerMapping,内部聚合:FilteringWebHandler/RouteDefinitionRouteLocator.<font color='red'>它是HandlerMapping的实现子类(WebFlux)</font>    
> 6. <font color='red'>重点:FilteringWebHandler和RoutePredicateHandlerMapping应该是入口了.</font>
