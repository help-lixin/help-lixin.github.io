---
layout: post
title: 'Spring Cloud Gateway 是如何结合Eureka做到服务发现的?(十)'
date: 2020-12-16
author: 李新
tags: SpringCloudGateway源码
---

### (1). Spring Cloud Gateway结合Eureka实现服务发现的配置
> 1. 根据配置信息(spring.cloud.gateway.discovery.locator).   
> 2. 寻找相应的配置类:(DiscoveryLocatorProperties).   
> 3. 根据DiscoveryLocatorProperties反向查找,在哪里有使用这个类(GatewayDiscoveryClientAutoConfiguration). 

```
spring:
  application:
    name: gateway-server  # 应用名称
  cloud:
    gateway:
      discovery:
          locator:
              # 是否与服务发现组件进行结合,通过serviceId转发到具体服务实例上.
              enabled: true                  # 是否开启基于服务发现的路由规则
              lower-case-service-id: true    # 是否将服务名称小写
```
### (2). DiscoveryLocatorProperties
> 结合服务发现的业务模型对象(DiscoveryLocatorProperties). 

```
package org.springframework.cloud.gateway.discovery;

@ConfigurationProperties("spring.cloud.gateway.discovery.locator")
public class DiscoveryLocatorProperties {
	// 是否启用服务发现
	private boolean enabled = false;
	// 
	private String routeIdPrefix;
	private String includeExpression = "true";
	private String urlExpression = "'lb://'+serviceId";
	//  是否将服务名称小写
	private boolean lowerCaseServiceId = false;

	// 断言列表
	private List<PredicateDefinition> predicates = new ArrayList<>();
	// 过滤器列表
	private List<FilterDefinition> filters = new ArrayList<>();
}
```
### (3). GatewayDiscoveryClientAutoConfiguration
> 重点在于这里:     
> 1. <font color='red'>注入了:EurekaDiscoveryClient,可以根据微服务的名称,获取到:对应的IP和端口.</font>
> 2. <font color='red'>PathRoutePredicateFactory:对Path进行正则路由.</font>     
> 3. <font color='red'>RewritePathGatewayFilterFactory:对Path进行重写.</font>   


```
package org.springframework.cloud.gateway.discovery;

@ConditionalOnProperty(name = "spring.cloud.gateway.enabled", matchIfMissing = true)
//  TODO ... ... 
public class GatewayDiscoveryClientAutoConfiguration {

	// ************************************************************************
	// 3.4 获取所有的路由定义信息(从DiscoveryClient中获取)
	// DiscoveryClientRouteDefinitionLocator属于:RouteDefinitionLocator的实现类.
	// RouteDefinitionLocator只有一个接口,返回所有的路由定义信息.
	// Flux<RouteDefinition> getRouteDefinitions();
	// ************************************************************************
	@Bean
	@ConditionalOnBean(DiscoveryClient.class)
	@ConditionalOnProperty(name = "spring.cloud.gateway.discovery.locator.enabled")
	public DiscoveryClientRouteDefinitionLocator discoveryClientRouteDefinitionLocator(
			// **************************************************************************
			// 3.5 注入:DiscoveryClient对象
			// org.springframework.cloud.client.discovery.composite.CompositeDiscoveryClient
			//     org.springframework.cloud.netflix.eureka.EurekaDiscoveryClient
			//     org.springframework.cloud.client.discovery.simple.SimpleDiscoveryClient
			// **************************************************************************
			DiscoveryClient discoveryClient, 
			DiscoveryLocatorProperties properties) {

		return new DiscoveryClientRouteDefinitionLocator(discoveryClient, properties);
	}// end discoveryClientRouteDefinitionLocator


	// ************************************************************************
	// 3.1 构建服务发现配置类(DiscoveryLocatorProperties)
	// 并初始化:断言列表和Filter
	// ************************************************************************
	@Bean
	public DiscoveryLocatorProperties discoveryLocatorProperties() {
		DiscoveryLocatorProperties properties = new DiscoveryLocatorProperties();
		properties.setPredicates(initPredicates());
		properties.setFilters(initFilters());
		return properties;
	}// end discoveryLocatorProperties

	// 初始化断言
	public static List<PredicateDefinition> initPredicates() {
		ArrayList<PredicateDefinition> definitions = new ArrayList<>();
		// TODO: add a predicate that matches the url at /serviceId?
		// add a predicate that matches the url at /serviceId/**
		PredicateDefinition predicate = new PredicateDefinition();
		// ***********************************************************************
		// 3.2 设置默认的断言工厂
		// PathRoutePredicateFactory
		// ***********************************************************************
		// name = path
		predicate.setName(normalizeRoutePredicateName(PathRoutePredicateFactory.class));
		predicate.addArg(PATTERN_KEY, "'/'+serviceId+'/**'");
		definitions.add(predicate);
		return definitions;
	} // end initPredicates

	//
	public static List<FilterDefinition> initFilters() {
		ArrayList<FilterDefinition> definitions = new ArrayList<>();
		// add a filter that removes /serviceId by default
		FilterDefinition filter = new FilterDefinition();

		// ***********************************************************************
		// 3.3 设置默认的过滤器
		// ***********************************************************************
		filter.setName(normalizeFilterFactoryName(RewritePathGatewayFilterFactory.class));
		String regex = "'/' + serviceId + '/(?<remaining>.*)'";
		String replacement = "'/${remaining}'";
		filter.addArg(REGEXP_KEY, regex);
		filter.addArg(REPLACEMENT_KEY, replacement);
		definitions.add(filter);

		return definitions;
	}// end initFilters
}
```
### (4). GatewayFilter
> 通过FilteringWebHandler类,查看请求有多少Filter(总共有九个).

```
org.springframework.cloud.gateway.filter.AdaptCachedBodyGlobalFilter
org.springframework.cloud.gateway.filter.NettyWriteResponseFilter
org.springframework.cloud.gateway.filter.ForwardPathFilter
org.springframework.cloud.gateway.filter.factory.RewritePathGatewayFilterFactory
org.springframework.cloud.gateway.filter.RouteToRequestUrlFilter
org.springframework.cloud.gateway.filter.LoadBalancerClientFilter
org.springframework.cloud.gateway.filter.WebsocketRoutingFilter
org.springframework.cloud.gateway.filter.NettyRoutingFilter
org.springframework.cloud.gateway.filter.ForwardRoutingFilter
```
### (5). RouteToRequestUrlFilter
```
public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
	// ************************************************************
	// 获取请求对应的路由信息(在断言阶段就已经赋值了.)
	// ************************************************************
	Route route = exchange.getAttribute(GATEWAY_ROUTE_ATTR);
	if (route == null) {
		return chain.filter(exchange);
	}
	log.trace("RouteToRequestUrlFilter start");
	URI uri = exchange.getRequest().getURI();
	boolean encoded = containsEncodedParts(uri);
	URI routeUri = route.getUri();

	if (hasAnotherScheme(routeUri)) {
		// this is a special url, save scheme to special attribute
		// replace routeUri with schemeSpecificPart
		exchange.getAttributes().put(GATEWAY_SCHEME_PREFIX_ATTR, routeUri.getScheme());
		routeUri = URI.create(routeUri.getSchemeSpecificPart());
	}

	if("lb".equalsIgnoreCase(routeUri.getScheme()) && routeUri.getHost() == null) {
		//Load balanced URIs should always have a host.  If the host is null it is most
		//likely because the host name was invalid (for example included an underscore)
		throw new IllegalStateException("Invalid host: " + routeUri.toString());
	}

	// **************************************************************
	// 
	// **************************************************************
	URI mergedUrl = UriComponentsBuilder.fromUri(uri)
			// .uri(routeUri)
			.scheme(routeUri.getScheme())
			// ************************************************************
			// lb://test-provider
			// ************************************************************
			.host(routeUri.getHost())
			.port(routeUri.getPort())
			.build(encoded)
			.toUri();

	// ***********************************************************************************		
	// 保存变量到exchange里,在LoadBalancerClientFilter会需要该变量
    // key = org.springframework.cloud.gateway.support.ServerWebExchangeUtils.gatewayRequestUrl
	// value = lb://TEST-PROVIDER/hello
	// ***********************************************************************************
	exchange.getAttributes().put(GATEWAY_REQUEST_URL_ATTR, mergedUrl);
	return chain.filter(exchange);
}// end
```
### (6). LoadBalancerClientFilter
```
// 负载均衡客户端:
protected final LoadBalancerClient loadBalancer;

private LoadBalancerProperties properties;

public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {

	// 在RouteToRequestUrlFilter里设置到exchange中的变量.
	// url = lb://TEST-PROVIDER/hello
	URI url = exchange.getAttribute(GATEWAY_REQUEST_URL_ATTR);

	String schemePrefix = exchange.getAttribute(GATEWAY_SCHEME_PREFIX_ATTR);
	if (url == null || (!"lb".equals(url.getScheme()) && !"lb".equals(schemePrefix))) {
		return chain.filter(exchange);
	}
	//preserve the original url
	addOriginalRequestUrl(exchange, url);


	log.trace("LoadBalancerClientFilter url before: " + url);


	// *****************************************************************
	// 调用负载均衡,根据微服务名称,获得一个微服务对应的实例信息.
	// *****************************************************************
	final ServiceInstance instance = choose(exchange);

	if (instance == null) {
		String msg = "Unable to find instance for " + url.getHost();
		if(properties.isUse404()) {
			throw new FourOFourNotFoundException(msg);
		}
		throw new NotFoundException(msg);
	}

	// uri = http://localhost:9000/hello
	URI uri = exchange.getRequest().getURI();

	
	String overrideScheme = instance.isSecure() ? "https" : "http";
	if (schemePrefix != null) {
		overrideScheme = url.getScheme();
	}

	// 重构URL
	// http://lixin-macbook:8080/hello
	URI requestUrl = loadBalancer.reconstructURI(new DelegatingServiceInstance(instance, overrideScheme), uri);

	log.trace("LoadBalancerClientFilter url chosen: " + requestUrl);
	

	exchange.getAttributes().put(GATEWAY_REQUEST_URL_ATTR, requestUrl);
	return chain.filter(exchange);
}
```
### (7). 总结
> PathRoutePredicateFactory负载断言.    
> RewritePathGatewayFilterFactory负责对正则表达式替换.   
> LoadBalancerClientFilter最终负责:lb://test-provider替换成:http://localhost:8080.   
