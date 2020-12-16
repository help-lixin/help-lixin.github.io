---
layout: post
title: 'Spring Cloud Gateway 自定义GlobalFilter(十七)'
date: 2020-12-15
author: 李新
tags: SpringCloudGateway
---

### (1). 自定义网关
> 需要实现:GlobalFilter和Ordered.  

### (2). CustomerGlobalFilter
```
package help.lixin.filter;

import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;

import reactor.core.publisher.Mono;

// 让Spring能自动识别.
@Component
public class CustomerGlobalFilter implements GlobalFilter, Ordered {

	@Override
	public int getOrder() {
		return 0;
	}

	@Override
	public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
		// TODO 业务逻辑处
		ServerHttpRequest request = exchange.getRequest();
		System.out.println("全局网关执行URI:" + request.getURI() + " Method: " + request.getMethodValue());
		return chain.filter(exchange); // 继续向下执行
	}
}

```
### (3). 测试
```
# 路由正常
curl  http://localhost:9000/consumer
consumer...Hello World!!!
```
### (4). 查看控制台
```
# 全局网关已经生效.
全局网关执行URI:http://localhost:9000/consumer Method: GET
自定义网关执行URI: http://localhost:9000/consumer
```