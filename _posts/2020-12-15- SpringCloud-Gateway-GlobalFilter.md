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

import java.io.Serializable;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.core.io.buffer.DataBuffer;
import org.springframework.http.HttpStatus;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.http.server.reactive.ServerHttpResponse;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;

import reactor.core.publisher.Mono;

@Component
public class CustomerGlobalFilter implements GlobalFilter, Ordered {

	private Logger logger = LoggerFactory.getLogger(CustomerGlobalFilter.class);

	@Override
	public int getOrder() {
		return 0;
	}

	@Override
	public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
		// TODO 业务逻辑处理.
		ServerHttpRequest request = exchange.getRequest();
		ServerHttpResponse response = exchange.getResponse();
		// 获得token
		String token = getToken(request);
		// 如果获取不到token
		if (null == token) {
			logger.warn("token is null");
			// 设置响应的协议头信息.
			response.getHeaders().add("Content-Type", "application/json");
			response.setStatusCode(HttpStatus.UNAUTHORIZED);

			// 创建返回信息体
			Response res = new Response();
			res.setMsg(HttpStatus.UNAUTHORIZED.getReasonPhrase());

			ObjectMapper objectMapper = new ObjectMapper();
			String body = "";
			try {
				// 将返回信息体转换成字符串
				body = objectMapper.writeValueAsString(res);
			} catch (JsonProcessingException e) {
			}

			// 将字符串转换成:DataBuffer
			DataBuffer buffer = response.bufferFactory().wrap(body.getBytes());
			// 拦截请求,不再继续往下执行.
			return response.writeWith(Mono.just(buffer));
		}
		logger.info("token validate success.");
		return chain.filter(exchange); // 继续向下执行
	}

	public String getToken(ServerHttpRequest request) {
		String token = null;
		// 从协议头里拿到:token
		token = request.getHeaders().getFirst("token");
		if (null == token) { // 拿不到的话,再从:params中拿
			token = request.getQueryParams().getFirst("token");
		}
		return token;
	}

	class Response implements Serializable {
		private static final long serialVersionUID = -7672972125381074209L;
		private String msg;

		public String getMsg() {
			return msg;
		}

		public void setMsg(String msg) {
			this.msg = msg;
		}
	}
}

```
### (3). 测试
```
# 路由正常(协议头添加token)
curl -H "token:abc123"  http://localhost:9000/consumer
consumer...Hello World!!!

# 路由正常(参数中添加token)
curl  http://localhost:9000/consumer?token=123
consumer...Hello World!!!

# 路由失败(没有添加token)
curl  http://localhost:9000/consumer
{"msg":"Unauthorized"}
```
