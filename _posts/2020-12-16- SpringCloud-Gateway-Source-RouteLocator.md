---
layout: post
title: 'Spring Cloud Gateway RouteLocator(五)'
date: 2020-12-16
author: 李新
tags: SpringCloudGateway源码
---

### (1). RouteLocator
> RouteDefinitionRouteLocator类的本质,逃离不了:RouteLocator接口的定义,应该是把:RouteDefinition中的数据,转换成:Route对象.    

```
public interface RouteLocator {
	// 仅此一个方法,获取多个Route
	Flux<Route> getRoutes();
}
```
### (2). Route
> RouteDefinition负责存储用户配置的配置,最终会被解析成:Route对象.

```
public class Route implements Ordered {
	private final String id;
	private final URI uri;
	private final int order;
	// predicate
	private final AsyncPredicate<ServerWebExchange> predicate;
	// filter
	private final List<GatewayFilter> gatewayFilters;
}
```
### (3). RouteDefinitionRouteLocator初始化
```
public class RouteDefinitionRouteLocator implements 
	         RouteLocator, 
	         BeanFactoryAware, 
	         ApplicationEventPublisherAware {

    public static final String DEFAULT_FILTERS = "defaultFilters";
	// application.yml配置的routes内容
	private final RouteDefinitionLocator routeDefinitionLocator;
	
	private final ConversionService conversionService;


	// key:  RoutePredicateFactory实现类的名称(替换:RoutePredicateFactory)
	//       例如:PathRoutePredicateFactory处理后:Path
	// value: RoutePredicateFactory实现类
	private final Map<String, RoutePredicateFactory> predicates = new LinkedHashMap<>();

	// key : GatewayFilterFactory实现类的名称(替换:GatewayFilterFactory),
	//       例如:RemoveRequestHeaderGatewayFilterFactory处理后:RemoveRequestHeader
	private final Map<String, GatewayFilterFactory> gatewayFilterFactories = new HashMap<>();

	// 配置文件(spring.cloud.gateway)
	private final GatewayProperties gatewayProperties;
	// Spel表达式
	private final SpelExpressionParser parser = new SpelExpressionParser();
	// BeanFactory工厂
	private BeanFactory beanFactory;

	public RouteDefinitionRouteLocator(
		           RouteDefinitionLocator routeDefinitionLocator,
				   List<RoutePredicateFactory> predicates,
				   List<GatewayFilterFactory> gatewayFilterFactories,GatewayProperties gatewayProperties,
				   ConversionService conversionService) {
        
		this.routeDefinitionLocator = routeDefinitionLocator;
		this.conversionService = conversionService;
		// 3.1 initFactories
		initFactories(predicates);
		// 3.2 gatewayFilterFactories 处理
		// RemoveRequestHeaderGatewayFilterFactory
		// key: RemoveRequestHeader
		// value: RemoveRequestHeaderGatewayFilterFactory

		gatewayFilterFactories.forEach(factory -> this.gatewayFilterFactories.put(factory.name(), factory));

		this.gatewayProperties = gatewayProperties;
	} // end 


	// 3.1 对RoutePredicateFactory进行处理.
	private void initFactories(List<RoutePredicateFactory> predicates) {
		predicates.forEach(factory -> {
			// PathRoutePredicateFactory
			// key = Path
			String key = factory.name();
			
			
			if (this.predicates.containsKey(key)) {
				this.logger.warn("A RoutePredicateFactory named "+ key
						+ " already exists, class: " + this.predicates.get(key)
						+ ". It will be overwritten.");
			}

			// key = Path
			// value = PathRoutePredicateFactory
			this.predicates.put(key, factory);

			if (logger.isInfoEnabled()) {
				logger.info("Loaded RoutePredicateFactory [" + key + "]");
			}
		});
	}// end initFactories

}
```
### (4). RouteDefinitionRouteLocator.getRoutes
```
public Flux<Route> getRoutes() {
	// 获取配置信息
	// routeDefinitionLocator = spring.cloud.gateway.routes
	return this.routeDefinitionLocator.getRouteDefinitions()
			// ******************************************************
			// 4.1 转换RouteDefinition为:Route对象
			// ******************************************************
			.map(this::convertToRoute)
			//TODO: error handling
			.map(route -> {
				if (logger.isDebugEnabled()) {
					logger.debug("RouteDefinition matched: " + route.getId());
				}
				return route;
			});
} // end getRoutes

// 4.1 转换RouteDefinition为:Route对象
private Route convertToRoute(RouteDefinition routeDefinition) {
	// 对Predicate部份进行处理
	AsyncPredicate<ServerWebExchange> predicate = combinePredicates(routeDefinition);
	// ************************************************
	// 对:GatewayFilter部份进行处理.
	// ************************************************
	List<GatewayFilter> gatewayFilters = getFilters(routeDefinition);

	// 最后构建成:Route对象
	return Route.async(routeDefinition)
			.asyncPredicate(predicate)
			.replaceFilters(gatewayFilters)
			.build();
} // end convertToRoute


private List<GatewayFilter> getFilters(RouteDefinition routeDefinition) {
	List<GatewayFilter> filters = new ArrayList<>();

	//TODO: support option to apply defaults after route specific filters?
	if (!this.gatewayProperties.getDefaultFilters().isEmpty()) {
		// *************************************************
		// 默认Filter
		// *************************************************
		filters.addAll(loadGatewayFilters(DEFAULT_FILTERS,
				this.gatewayProperties.getDefaultFilters()));
	}

	if (!routeDefinition.getFilters().isEmpty()) {
		// *************************************************
		// 挨个添加:filter,会对名称对应的filter对象给加载进来.
		// *************************************************
		filters.addAll(loadGatewayFilters(routeDefinition.getId(), routeDefinition.getFilters()));
	}

	AnnotationAwareOrderComparator.sort(filters);
	return filters;
} // end getFilters

private List<GatewayFilter> loadGatewayFilters(String id, List<FilterDefinition> filterDefinitions) {
	List<GatewayFilter> filters = filterDefinitions.stream()
			.map(definition -> {
				GatewayFilterFactory factory = this.gatewayFilterFactories.get(definition.getName());
				if (factory == null) {
					throw new IllegalArgumentException("Unable to find GatewayFilterFactory with name " + definition.getName());
				}
				Map<String, String> args = definition.getArgs();
				if (logger.isDebugEnabled()) {
					logger.debug("RouteDefinition " + id + " applying filter " + args + " to " + definition.getName());
				}

				Map<String, Object> properties = factory.shortcutType().normalize(args, factory, this.parser, this.beanFactory);

				Object configuration = factory.newConfig();

				ConfigurationUtils.bind(configuration, properties, factory.shortcutFieldPrefix(),
						definition.getName(), validator, conversionService);

				GatewayFilter gatewayFilter = factory.apply(configuration);
				if (this.publisher != null) {
					this.publisher.publishEvent(new FilterArgsEvent(this, id, properties));
				}
				return gatewayFilter;
			})
			.collect(Collectors.toList());

	ArrayList<GatewayFilter> ordered = new ArrayList<>(filters.size());
	for (int i = 0; i < filters.size(); i++) {
		GatewayFilter gatewayFilter = filters.get(i);
		if (gatewayFilter instanceof Ordered) {
			ordered.add(gatewayFilter);
		}
		else {
			ordered.add(new OrderedGatewayFilter(gatewayFilter, i + 1));
		}
	}
	return ordered;
} //end loadGatewayFilters
```
### (5). 总结
> RouteDefinitionRouteLocator负责解析:RouteDefinitionLocator(spring.cloud.gateway.routes),并转换成:Route.  
