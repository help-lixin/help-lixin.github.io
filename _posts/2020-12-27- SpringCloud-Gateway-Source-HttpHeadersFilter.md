---
layout: post
title: 'Spring Cloud Gateway 如何为Request(Response)增加自定义协议头呢?(十一)'
date: 2020-12-27
author: 李新
tags: SpringCloudGateway源码
---

### (1). 如何为Request(Response)增加自定义协议头呢?
> 在Spring Cloud Gateway中有一个比较重要的NettyRoutingFilter它是GlobalFilter的实现类.这个类承接最终的请求转发.  

### (2). 看下NettyRoutingFilter类信息
```
public class NettyRoutingFilter implements GlobalFilter, Ordered {

	private final HttpClient httpClient;
	private final ObjectProvider<List<HttpHeadersFilter>> headersFilters;
	private final HttpClientProperties properties;


	public NettyRoutingFilter(HttpClient httpClient,
	                          // ***************************************************
							  // 3. 构造器,依赖:HttpHeadersFilter
							  // ***************************************************
							  ObjectProvider<List<HttpHeadersFilter>> headersFilters,
							  HttpClientProperties properties) {
		this.httpClient = httpClient;
		this.headersFilters = headersFilters;
		this.properties = properties;
	}
	
	
	public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
		
		// ... ...
		// 为request添加Header
		HttpHeaders filtered = filterRequest(this.headersFilters.getIfAvailable(),exchange);
		final DefaultHttpHeaders httpHeaders = new DefaultHttpHeaders();
		filtered.forEach(httpHeaders::set);
        // ... ...
		
		Flux<HttpClientResponse> responseFlux = 
		this.httpClient
		    .chunkedTransfer(chunkedTransfer)
			.request(method)
			.uri(url)
			.send((req, nettyOutbound) -> {
				// **********************************************
				// 为request配置header
				// **********************************************
				req.headers(httpHeaders);
				// ... ...
			})
			.responseConnection((res, connection) -> {
				ServerHttpResponse response = exchange.getResponse();
				
				// 创建response headers
				// put headers and status so filters can modify the response
				HttpHeaders headers = new HttpHeaders();
				
				// 遍历response里所有的header,添加到:headers
				res.responseHeaders().forEach(entry -> headers.add(entry.getKey(), entry.getValue()));


				// **********************************************
				// 遍历依赖的:HttpHeaders,可增加或删除Header
				// **********************************************
				HttpHeaders filteredResponseHeaders = HttpHeadersFilter.filter(
						this.headersFilters.getIfAvailable(), headers, exchange, Type.RESPONSE);

				// 为response添加header
				response.getHeaders().putAll(filteredResponseHeaders);
				
			});
			// ... ... 
	}
}
```
### (3). 总结
> 如果要为request或者response增加协议信息,只需要实现:HttpHeadersFilter类,并实现(supports/filter)即可.   