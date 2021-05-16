---
layout: post
title: 'Spring Cloud Gateway RemoveResponseHeaderGatewayFilterFactory(十一)'
date: 2020-12-16
author: 李新
tags: SpringCloudGateway源码
---

### (1). 概述
> 有这样一个需求:期望在用户请求之前,添加一个唯一requestId,然后,把这个唯一rquestId返回(Response).

### (2). 研究下源码(RemoveResponseHeaderGatewayFilterFactory)
```
public class RemoveResponseHeaderGatewayFilterFactory
		extends AbstractGatewayFilterFactory<AbstractGatewayFilterFactory.NameConfig> {

	public RemoveResponseHeaderGatewayFilterFactory() {
		super(NameConfig.class);
	}

	@Override
	public List<String> shortcutFieldOrder() {
		return Arrays.asList(NAME_KEY);
	}

	@Override
	public GatewayFilter apply(NameConfig config) {
		// *****************************************************
		// 1.自定义:GatewayFilter
		// *****************************************************
		return new GatewayFilter() {
			@Override
			public Mono<Void> filter(ServerWebExchange exchange,
					GatewayFilterChain chain) {
				// *****************************************************
				// 2.对返回(Response)协议头进行更改
				// *****************************************************
				return chain.filter(exchange).then(Mono.fromRunnable(() -> exchange
						.getResponse().getHeaders().remove(config.getName())));
			}

			@Override
			public String toString() {
				return filterToStringCreator(
						RemoveResponseHeaderGatewayFilterFactory.this)
								.append("name", config.getName()).toString();
			}
		};
	}
}

```
### (3). 总结
> 我感觉好像没什么要总结的.