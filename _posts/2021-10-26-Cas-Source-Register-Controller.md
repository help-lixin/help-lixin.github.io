---
layout: post
title: 'CAS源码之向Spring MVC中手工注册Controller(七)' 
date: 2021-10-26
author: 李新
tags:  CAS
---

### (1). 概述
在看Cas源码时,无意中发现这个,手工向Spring容器中注册Controller,所以,特意记录下来.
### (2). 手工定义Controller
```
// 1. 定义一个Controller(bean名称为:rootController)
@Bean
@ConditionalOnMissingBean(name = "rootController")
public Controller rootController() {
	return new ParameterizableViewController() {
		@Override
		protected ModelAndView handleRequestInternal(final HttpServletRequest request,
													 final HttpServletResponse response) {
			val queryString = request.getQueryString();
			// 应用上下文()
			val url = request.getContextPath() + "/login"
				+ Optional.ofNullable(queryString).map(string -> '?' + string).orElse(StringUtils.EMPTY);
			return new ModelAndView(new RedirectView(response.encodeURL(url)));
		}
	};
}
```
### (3). 向Spring MVC中注册Bean(Controller)
```
@Configuration(value = "casWebAppConfiguration", proxyBeanMethods = false)
@EnableConfigurationProperties(CasConfigurationProperties.class)
// 1. 实现:WebMvcConfigurer
public class CasWebAppConfiguration implements WebMvcConfigurer {
	
	// 2. 定义HandlerMapping
	@Bean
	@Autowired
	public SimpleUrlHandlerMapping handlerMapping(@Qualifier("rootController") final Controller rootController) {
		val mapping = new SimpleUrlHandlerMapping();

		mapping.setOrder(1);
		mapping.setAlwaysUseFullPath(true);
		mapping.setRootHandler(rootController);
		val urls = new HashMap<String, Object>();
		urls.put("/", rootController);

		mapping.setUrlMap(urls);
		return mapping;
	}

	// 3. DispatcherServlet配置拦截所有的请求
	@Override
	public void addInterceptors(final InterceptorRegistry registry) {
		registry.addInterceptor(localeChangeInterceptor.getObject()).addPathPatterns("/**");
	} // end addInterceptors
	
}	
```
### (4). 总结
> 今天无意中发现,通过这种手工的方式,也是可以向Spring容器中注册Controller和HandlerMapping.  