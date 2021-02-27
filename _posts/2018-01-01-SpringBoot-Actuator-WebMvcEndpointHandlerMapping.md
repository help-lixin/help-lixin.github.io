---
layout: post
title: 'Spring Boot Actuator(WebMvcEndpointHandlerMapping)'
date: 2018-01-01
author: 李新
tags: SpringBoot
---

### (1). 概述
> 如果我们期望针对:Endpoint的请求都进行鉴权,该如何设计呢?   
> Endpoint实际是有一套自己的注解,然后,把这套注解转换成业务模型,并向SpringMVC靠扰.  

### (2). WebMvcEndpointManagementContextConfiguration
```
@ManagementContextConfiguration
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass(DispatcherServlet.class)
@ConditionalOnBean({ DispatcherServlet.class, WebEndpointsSupplier.class })
@EnableConfigurationProperties(CorsEndpointProperties.class)
public class WebMvcEndpointManagementContextConfiguration {

	@Bean
	// *************************************************************
	// 重点:Spring设计的优势就在于这里,可以允许你写重掉这个Bean
	// 当容器中,不存在这个Bean时,这个配置才会生效,所以,你自己写一个Bean就好了,就能让这里的逻辑失效.
	// *************************************************************
	@ConditionalOnMissingBean
	// *************************************************************
	// 你可能有这样的需求:想对endpoint进行鉴权.那么怎么做呢?
	// 你可以写重这部份的逻辑.
	// *************************************************************
	// WebMvcEndpointHandlerMapping XXXHandlerMapping属于Spring MVC中的一部份.
	public WebMvcEndpointHandlerMapping webEndpointServletHandlerMapping(
			WebEndpointsSupplier webEndpointsSupplier,
			ServletEndpointsSupplier servletEndpointsSupplier,
			ControllerEndpointsSupplier controllerEndpointsSupplier,
			EndpointMediaTypes endpointMediaTypes, CorsEndpointProperties corsProperties,
			WebEndpointProperties webEndpointProperties) {
		List<ExposableEndpoint<?>> allEndpoints = new ArrayList<>();
		Collection<ExposableWebEndpoint> webEndpoints = webEndpointsSupplier
				.getEndpoints();
		allEndpoints.addAll(webEndpoints);
		allEndpoints.addAll(servletEndpointsSupplier.getEndpoints());
		allEndpoints.addAll(controllerEndpointsSupplier.getEndpoints());
		EndpointMapping endpointMapping = new EndpointMapping(
				webEndpointProperties.getBasePath());
		return new WebMvcEndpointHandlerMapping(endpointMapping, webEndpoints,
				endpointMediaTypes, corsProperties.toCorsConfiguration(),
				new EndpointLinksResolver(allEndpoints,
						webEndpointProperties.getBasePath()));
	}
	
	// ... ...
}
```
### (3). 自定义WebMvcEndpointHandlerMapping
```
class WebMvcEndpointExtHandlerMapping extends WebMvcEndpointHandlerMapping {

	public WebMvcEndpointExtHandlerMapping(EndpointMapping endpointMapping, Collection<ExposableWebEndpoint> endpoints,
			EndpointMediaTypes endpointMediaTypes, CorsConfiguration corsConfiguration,
			EndpointLinksResolver linksResolver) {
		super(endpointMapping, endpoints, endpointMediaTypes, corsConfiguration, linksResolver);
	}

	@Override
	protected void extendInterceptors(List<Object> interceptors) {
		super.extendInterceptors(interceptors);
		// **************************************************
		// 添加你自定义的拦截器即可
		// **************************************************
		interceptors.add(new CustomHandlerInterceptor());
	}
} // WebMvcEndpointExtHandlerMapping

class CustomHandlerInterceptor implements HandlerInterceptor {
	public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {
		System.out.println("=================================================");
		return true;
	}
}// end CustomHandlerInterceptor
```
### (4). 配置WebMvcEndpointHandlerMapping
```
@Bean
public WebMvcEndpointHandlerMapping webEndpointServletHandlerMapping(
		WebEndpointsSupplier webEndpointsSupplier,
		ServletEndpointsSupplier servletEndpointsSupplier,
		ControllerEndpointsSupplier controllerEndpointsSupplier,
		EndpointMediaTypes endpointMediaTypes, CorsEndpointProperties corsProperties,
		WebEndpointProperties webEndpointProperties) {
	List<ExposableEndpoint<?>> allEndpoints = new ArrayList<ExposableEndpoint<?>>();
	Collection<ExposableWebEndpoint> webEndpoints = webEndpointsSupplier
			.getEndpoints();
	allEndpoints.addAll(webEndpoints);
	allEndpoints.addAll(servletEndpointsSupplier.getEndpoints());
	allEndpoints.addAll(controllerEndpointsSupplier.getEndpoints());
	EndpointMapping endpointMapping = new EndpointMapping(
			webEndpointProperties.getBasePath());
	// **********************************************************		
	// 使用自定义:WebMvcEndpointExtHandlerMapping
	// **********************************************************		
	return new WebMvcEndpointExtHandlerMapping(endpointMapping, webEndpoints,
			endpointMediaTypes, corsProperties.toCorsConfiguration(),
			new EndpointLinksResolver(allEndpoints,
					webEndpointProperties.getBasePath()));
}
```
### (5). 总结
> Endpoint无法不过是把SpringMVC那一套的注解,变成它自己的那一套注解.  
> 1. 解析自己的注解,转换成业务模型.    
> 2. 把业务模型的信息,向SpringMVC进行靠扰(比如:注册HandlerMapping...)   