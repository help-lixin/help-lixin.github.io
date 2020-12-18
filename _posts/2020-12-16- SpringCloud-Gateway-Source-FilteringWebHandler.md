---
layout: post
title: 'Spring Cloud Gateway FilteringWebHandler(七)'
date: 2020-12-16
author: 李新
tags: SpringCloudGateway源码
---

### (1). 概述
> 在前面(第六节),分析了:RoutePredicateHandlerMapping,它主要获取用户配置:Route中的断言(Predicate)对象.调用apply(Exchange).获取符合断言的Route对象.并把这个Route设置到:Exchange中.所以:Gateway是首先执行断言(Predicate).      
> 这一节,主要讲解:Filter的执行.    

### (2). FilteringWebHandler类图
> 1. 所有的:GlobalFilter都会进行适配成:GatewayFilterAdapter(典型的适配模式),而GatewayFilterAdapter,又会被:OrderedGatewayFilter对象包裹.   

!["FilteringWebHandler"](/assets/spring-cloud-gateway/imgs/spring-cloud-gateway-filtering-web-handler-class.jpg)

### (2). FilteringWebHandler
```
public class FilteringWebHandler 
	    // ***********************************************
       // SimpleHandlerAdapter会回调handler方法.
	   // ***********************************************
       implements WebHandler {

    private final List<GatewayFilter> globalFilters;

	public FilteringWebHandler(List<GlobalFilter> globalFilters) {
		// 2.1 对globalFilters进行加工处理
		this.globalFilters = loadFilters(globalFilters);
	}

	private static List<GatewayFilter> loadFilters(List<GlobalFilter> filters) {
		// 2.2 适配并加工
		return filters.stream()
				.map(filter -> {
					// **********************************************
					// 适配模式,把GlobalFilter适配成:GatewayFilter
					// **********************************************
					GatewayFilterAdapter gatewayFilter = new GatewayFilterAdapter(filter);

					// 判断是否为:Ordered排序的子类
					if (filter instanceof Ordered) {
						int order = ((Ordered) filter).getOrder();
						// Wrapper模式
						return new OrderedGatewayFilter(gatewayFilter, order);
					}
					return gatewayFilter;
				}).collect(Collectors.toList());
	} // end loadFilters


	// reactor-netty解析后,最终会调用:handler.
	public Mono<Void> handle(ServerWebExchange exchange) {
		// 获取用户级别的:GatewayFilter
		// GATEWAY_ROUTE_ATTR是在:
		// RoutePredicateHandlerMapping.getHandlerInternal
		// 进行设置的.
		Route route = exchange.getRequiredAttribute(GATEWAY_ROUTE_ATTR);
		List<GatewayFilter> gatewayFilters = route.getFilters();

		// 获取全局级别的:GatewayFilter(GlobalFilter已经被适配成:GatewayFilter)
		List<GatewayFilter> combined = new ArrayList<>(this.globalFilters);
		combined.addAll(gatewayFilters);
		//TODO: needed or cached?
		// 排序
		AnnotationAwareOrderComparator.sort(combined);

		if (logger.isDebugEnabled()) {
			logger.debug("Sorted gatewayFilterFactories: "+ combined);
		}

		//  **************************************************************************
		// RouteToRequestUrlFilter负责把网关:URL进行转换成target uri.
		// org.springframework.cloud.gateway.filter.RouteToRequestUrlFilter

		// NettyRoutingFilter负责把请求转发到:target uri
		// org.springframework.cloud.gateway.filter.NettyRoutingFilter
		//  **************************************************************************

		// combined = 
		// [
			OrderedGatewayFilter{delegate=GatewayFilterAdapter{delegate=AdaptCachedBodyGlobalFilter}, order=-2147482648}, 
			OrderedGatewayFilter{delegate=GatewayFilterAdapter{delegate=NettyWriteResponseFilter}, order=-1}, 
			OrderedGatewayFilter{delegate=GatewayFilterAdapter{delegate=ForwardPathFilter}, order=0}, 
			OrderedGatewayFilter{delegate=RewritePathGatewayFilterFactory$$Lambda$570/369895363, order=1}, 
			OrderedGatewayFilter{delegate=SetStatusGatewayFilterFactory$$Lambda$572/14733990, order=2}, 
			OrderedGatewayFilter{delegate=GatewayFilterAdapter{delegate=RouteToRequestUrlFilter}, order=10000}, 
			OrderedGatewayFilter{delegate=GatewayFilterAdapter{delegate=WebsocketRoutingFilter}, order=2147483646}, 
			OrderedGatewayFilter{delegate=GatewayFilterAdapter{NettyRoutingFilter}, order=2147483647}, 
			OrderedGatewayFilter{delegate=GatewayFilterAdapter{delegate=ForwardRoutingFilter}, order=2147483647}
		//	]
		// 挨个调用相应的:filter方法
		return new DefaultGatewayFilterChain(combined).filter(exchange);
	}// end handle

}
```

### (3). NettyRoutingFilter初始化
> NettyRoutingFilter负责proxy请求到目标服务器.

```
public class GatewayAutoConfiguration {
	@Configuration
	@ConditionalOnClass(HttpClient.class)
	protected static class NettyConfiguration {
		@Bean
		@ConditionalOnMissingBean
		public HttpClient httpClient(HttpClientProperties properties) {

			// configure pool resources
			HttpClientProperties.Pool pool = properties.getPool();

			ConnectionProvider connectionProvider;
			if (pool.getType() == DISABLED) {
				connectionProvider = ConnectionProvider.newConnection();
			} else if (pool.getType() == FIXED) {
				connectionProvider = ConnectionProvider.fixed(pool.getName(),
						pool.getMaxConnections(), pool.getAcquireTimeout());
			} else {
				connectionProvider = ConnectionProvider.elastic(pool.getName());
			}

			HttpClient httpClient = HttpClient.create(connectionProvider)
				.tcpConfiguration(tcpClient -> {

					if (properties.getConnectTimeout() != null) {
						tcpClient = tcpClient.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, properties.getConnectTimeout());
					}

					// configure proxy if proxy host is set.
					HttpClientProperties.Proxy proxy = properties.getProxy();

					if (StringUtils.hasText(proxy.getHost())) {

						tcpClient = tcpClient.proxy(proxySpec -> {
							ProxyProvider.Builder builder = proxySpec
									.type(ProxyProvider.Proxy.HTTP)
									.host(proxy.getHost());

							PropertyMapper map = PropertyMapper.get();

							map.from(proxy::getPort)
									.whenNonNull()
									.to(builder::port);
							map.from(proxy::getUsername)
									.whenHasText()
									.to(builder::username);
							map.from(proxy::getPassword)
									.whenHasText()
									.to(password -> builder.password(s -> password));
							map.from(proxy::getNonProxyHostsPattern)
									.whenHasText()
									.to(builder::nonProxyHosts);
						});
					}
					return tcpClient;
				});

			HttpClientProperties.Ssl ssl = properties.getSsl();
			if (ssl.getTrustedX509CertificatesForTrustManager().length > 0
					|| ssl.isUseInsecureTrustManager()) {
				httpClient = httpClient.secure(sslContextSpec -> {
					// configure ssl
					SslContextBuilder sslContextBuilder = SslContextBuilder.forClient();

					X509Certificate[] trustedX509Certificates = ssl
							.getTrustedX509CertificatesForTrustManager();
					if (trustedX509Certificates.length > 0) {
						sslContextBuilder.trustManager(trustedX509Certificates);
					} else if (ssl.isUseInsecureTrustManager()) {
						sslContextBuilder.trustManager(InsecureTrustManagerFactory.INSTANCE);
					}

					sslContextSpec.sslContext(sslContextBuilder)
							.defaultConfiguration(ssl.getDefaultConfigurationType())
							.handshakeTimeout(ssl.getHandshakeTimeout())
							.closeNotifyFlushTimeout(ssl.getCloseNotifyFlushTimeout())
							.closeNotifyReadTimeout(ssl.getCloseNotifyReadTimeout());
				});
			}

			return httpClient;
		}

		@Bean
		public HttpClientProperties httpClientProperties() {
			return new HttpClientProperties();
		}

		// ***************************************************************
		// NettyRoutingFilter
		// ***************************************************************
		@Bean
		public NettyRoutingFilter routingFilter(HttpClient httpClient,
												ObjectProvider<List<HttpHeadersFilter>> headersFilters,
												HttpClientProperties properties) {
			return new NettyRoutingFilter(httpClient, headersFilters, properties);
		}

		@Bean
		public NettyWriteResponseFilter nettyWriteResponseFilter(GatewayProperties properties) {
			return new NettyWriteResponseFilter(properties.getStreamingMediaTypes());
		}

		@Bean
		public ReactorNettyWebSocketClient reactorNettyWebSocketClient(HttpClient httpClient) {
			return new ReactorNettyWebSocketClient(httpClient);
		}
	} //end NettyConfiguration
} 
```

### (4). 总结
> 1. FilteringWebHandler是WebHandler的实现类,它负责把GlobalFilter转换成:GatewayFilter.      
> 2. handler在执行时,获得配置的:GatewayFilter和全局GatewayFilterAdapter.调用相应的:filter方法.     

