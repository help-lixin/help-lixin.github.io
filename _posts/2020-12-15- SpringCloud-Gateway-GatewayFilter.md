---
layout: post
title: 'Spring Cloud Gateway GatewayFilter(十六)'
date: 2020-12-15
author: 李新
tags: SpringCloudGateway
---

### (1). 自定义网关
> 需要实现:GatewayFilter和Ordered.  

### (2). CustomerGatewayFilter
```
package help.lixin.filter;

import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.core.Ordered;
import org.springframework.web.server.ServerWebExchange;

import reactor.core.publisher.Mono;

public class CustomerGatewayFilter implements GatewayFilter, Ordered {

	@Override
	public int getOrder() {
		return 0;
	}

	@Override
	public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
		System.out.println(exchange.getRequest().getURI());
		// TODO 在此处可对exchange进行改变
		return chain.filter(exchange); // 继续向下执行
	}
}
```
### (3). 配置路由信息
```
package help.lixin.conf;

import java.util.Arrays;

import org.springframework.cloud.gateway.route.RouteLocator;
import org.springframework.cloud.gateway.route.builder.RouteLocatorBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import help.lixin.filter.CustomerGatewayFilter;

@Configuration
public class GatewayRoutesConfiguration {

	@Bean
	public RouteLocator routeLocator(RouteLocatorBuilder builder) {
		return builder //
				.routes() //
				.route(t ->
				// path
				t.path("/consumer/**")
                // uri
                .uri("http://localhost:7070/")
                // filters
                .filters(Arrays.asList(new CustomerGatewayFilter()))
                // id
                .id("test-consumer-service"))
				.build();
	}
}
```
### (4). application.yml(不需要再配置路由信息)
```
#端口
server:
  port: 9000
```
### (5). Application增加自动扫描
```
package help.lixin;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.SpringBootConfiguration;
import org.springframework.boot.autoconfigure.EnableAutoConfiguration;
import org.springframework.context.annotation.ComponentScan;

@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan
public class Application {
	public static void main(String[] args) {
		SpringApplication.run(Application.class, args);
	}
}

```
### (6). 测试
```
# 路由正确,再配合控制台查看内容
curl  http://localhost:9000/consumer
consumer...Hello World!!!

# 控制台打印内容:
自定义网关执行URI: http://localhost:9000/consumer
```