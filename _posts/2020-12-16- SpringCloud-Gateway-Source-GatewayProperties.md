---
layout: post
title: 'Spring Cloud Gateway GatewayProperties(一)'
date: 2020-12-16
author: 李新
tags: SpringCloudGateway源码
---

### (1). GatewayProperties类结构图

!["GatewayProperties"](/assets/spring-cloud-gateway/imgs/spring-cloud-gateway-GatewayProperties-class.jpg)

### (2). GatewayAutoConfiguration
```
@Bean
public GatewayProperties gatewayProperties() {
    return new GatewayProperties();
} // end gatewayProperties
```
### (3). GatewayProperties
> GatewayProperties包含多个:RouteDefinition和多个FilterDefinition

```
// 配置属性的名称
@ConfigurationProperties("spring.cloud.gateway")
public class GatewayProperties {
    private List<RouteDefinition> routes = new ArrayList<>();
    private List<FilterDefinition> defaultFilters = new ArrayList<>();
}
```
### (4). RouteDefinition
```
public class RouteDefinition {
   private String id = UUID.randomUUID().toString();
   // *****************************************************
   // predicate定义
   // *****************************************************
   private List<PredicateDefinition> predicates = new ArrayList<>();
   private List<FilterDefinition> filters = new ArrayList<>();
   private URI uri;
   private int order = 0;
}
```
### (5). PredicateDefinition
```
public class PredicateDefinition {
    private String name;
	private Map<String, String> args = new LinkedHashMap<>();

    // 构造该对象需要传递一个文本
    // 对文本按"等于"号进行分隔,变成map
    public PredicateDefinition(String text) {
		int eqIdx = text.indexOf('=');
		if (eqIdx <= 0) {
			throw new ValidationException("Unable to parse PredicateDefinition text '" + text + "'" +
					", must be of the form name=value");
		}
		setName(text.substring(0, eqIdx));
		String[] args = tokenizeToStringArray(text.substring(eqIdx+1), ",");

		for (int i=0; i < args.length; i++) {
            // *****************************************
            // 组装:Predicate
            // *****************************************
			this.args.put(NameUtils.generateName(i), args[i]);
		}
	}// end 构造器结束
}
```