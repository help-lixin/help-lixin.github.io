---
layout: post
title: 'Spring Cloud Gateway GatewayFilterFactory(二)'
date: 2020-12-16
author: 李新
tags: SpringCloudGateway源码
---

### (1). GatewayFilterFactory类结构图
> 从类的结构图可以分析出:GatewayFilterFactory是GatewayFilter的工厂类.   

!["GatewayFilterFactory类结构图"](/assets/spring-cloud-gateway/imgs/spring-cloud-gateway-gateway-filter-factory-class.jpg)

### (2). GatewayFilterFactory
```
public interface GatewayFilterFactory<C> 
       extends ShortcutConfigurable, 
               Configurable<C> {

    String NAME_KEY = "name";
	String VALUE_KEY = "value";

    default GatewayFilter apply(Consumer<C> consumer) {
		C config = newConfig();
		consumer.accept(config);
		return apply(config);
	}// end apply

    // 获取GatewayFilterFactory实现类的名称,并删除后缀
    default String name() {
        // getClass() = RemoveRequestHeaderGatewayFilterFactory
        // RemoveRequestHeader
		return NameUtils.normalizeRoutePredicateName(getClass());
	}// end name


    // ************************************************
    // 业务需要实现方法.
    // ************************************************
    GatewayFilter apply(C config);
}
```
### (3). 咱看一个简单的GatewayFilterFactory(RemoveRequestHeaderGatewayFilterFactory)
```
public class RemoveRequestHeaderGatewayFilterFactory 
       extends AbstractGatewayFilterFactory<AbstractGatewayFilterFactory.NameConfig> {
   
   // 设置使用的配置
   // - RemoveRequestHeader=token
   public RemoveRequestHeaderGatewayFilterFactory() {
		super(NameConfig.class);
   }

    public GatewayFilter apply(NameConfig config) {
        // jdk1.8的语法,创建了一个:GatewayFilter
		return (exchange, chain) -> {
			ServerHttpRequest request = exchange.getRequest().mutate()
                    // *****************************************************
                    // 删除指定的协义头名称
                    // *****************************************************
					.headers(httpHeaders -> httpHeaders.remove(config.getName()))
					.build();

			return chain.filter(exchange.mutate().request(request).build());
		};
	}// end apply
}
```
### (4). 总结
> GatewayFilterFactory是GatewayFilter的工厂类.是为了简化:GatewayFilterFactory开发提供的一个工厂类. 
