---
layout: post
title: 'Spring Cloud Gateway RoutePredicateFactory(三)'
date: 2020-12-16
author: 李新
tags: SpringCloudGateway源码
---

### (1). RoutePredicateFactory类结构图
!["RoutePredicateFactory类结构图"](/assets/spring-cloud-gateway/imgs/spring-cloud-gateway-route-predicate-factory-class.jpg)

### (2). RoutePredicateFactory
```
public interface RoutePredicateFactory<C> 
       extends ShortcutConfigurable, Configurable<C> {

    
    Predicate<ServerWebExchange> apply(C config);

}
```
### (3). BetweenRoutePredicateFactory
```
public class BetweenRoutePredicateFactory 
       extends AbstractRoutePredicateFactory<BetweenRoutePredicateFactory.Config> {
   
    public static final String DATETIME1_KEY = "datetime1";
	public static final String DATETIME2_KEY = "datetime2";

	public BetweenRoutePredicateFactory() {
		super(Config.class);
	}

    public Predicate<ServerWebExchange> apply(Config config) {
		ZonedDateTime datetime1 = config.datetime1;
		ZonedDateTime datetime2 = config.datetime2;
		Assert.isTrue(datetime1.isBefore(datetime2),
				config.datetime1 +
				" must be before " + config.datetime2);
        // JDK8语法,创建: Predicate
		return exchange -> {
			final ZonedDateTime now = ZonedDateTime.now();
			return now.isAfter(datetime1) && now.isBefore(datetime2);
		};
	} // end apply
}
```
### (4). 总结
> RoutePredicateFactory是RoutePredicate的工厂类,负责创建:RoutePredicate. 