---
layout: post
title: 'Spring Cloud Gateway RouteDefinitionLocator(九)'
date: 2020-12-16
author: 李新
tags: SpringCloudGateway源码
---

### (1). 概述
> RouteDefinitionLocator固名思意:即是路由定义的加载器.可以理解为:加载配置文件的策略.    

### (2). 查看RouteDefinitionLocator的实现类
> DiscoveryClientRouteDefinitionLocator  配合服务发现来实现动态配置.   
> CompositeRouteDefinitionLocator        固名思意:即组合模式.  
> CachingRouteDefinitionLocator          基于缓存的组合模式.支持RefreshRoutesEvent事件. 

!["RouteDefinitionLocator"](/assets/spring-cloud-gateway/imgs/spring-cloud-gateway-route-definition-locator.jpg)

### (3). 查看RouteDefinitionLocator的初始化过程
>  PropertiesRouteDefinitionLocator 和 InMemoryRouteDefinitionRepository.     
>  RouteDefinitionLocator只配置了两个,实际是可以组合,并缓存的.   

```
@Bean
@ConditionalOnMissingBean
public PropertiesRouteDefinitionLocator propertiesRouteDefinitionLocator(GatewayProperties properties) {
	return new PropertiesRouteDefinitionLocator(properties);
}

@Bean
@ConditionalOnMissingBean(RouteDefinitionRepository.class)
public InMemoryRouteDefinitionRepository inMemoryRouteDefinitionRepository() {
	return new InMemoryRouteDefinitionRepository();
}
```
### (4). PropertiesRouteDefinitionLocator
```
public class PropertiesRouteDefinitionLocator implements RouteDefinitionLocator {

	private final GatewayProperties properties;

	public PropertiesRouteDefinitionLocator(GatewayProperties properties) {
		this.properties = properties;
	}

	// 直接把:GatewayProperties.routes返回去,好简单的实现
	@Override
	public Flux<RouteDefinition> getRouteDefinitions() {
		return Flux.fromIterable(this.properties.getRoutes());
	}
}
```
### (5). InMemoryRouteDefinitionRepository
```
public class InMemoryRouteDefinitionRepository implements RouteDefinitionRepository {

	// 定义缓存
	private final Map<String, RouteDefinition> routes = 
	    synchronizedMap(new LinkedHashMap<String, RouteDefinition>());

	// 保存:RouteDefinition
	@Override
	public Mono<Void> save(Mono<RouteDefinition> route) {
		return route.flatMap( r -> {
			routes.put(r.getId(), r);
			return Mono.empty();
		});
	}

	// 根据routeID删除:RouteDefinition
	@Override
	public Mono<Void> delete(Mono<String> routeId) {
		return routeId.flatMap(id -> {
			if (routes.containsKey(id)) {
				routes.remove(id);
				return Mono.empty();
			}
			return Mono.defer(() -> Mono.error(new NotFoundException("RouteDefinition not found: "+routeId)));
		});
	}

	// 从缓存中获取所有的RouteDefinition.
	@Override
	public Flux<RouteDefinition> getRouteDefinitions() {
		return Flux.fromIterable(routes.values());
	}
}
```
### (6). 总结
> 如何实现Gatway不重启,重新加载配置文件呢?    
> 1. 从Spring容器中拿出:InMemoryRouteDefinitionRepository(因为该对象是单例的).
> 2. 执行你的业务操作(save/delete).  
> 3. 发布事件:RefreshRoutesEvent.触发:RouteLocator,清除所有已经触析好了的:Route.