---
layout: post
title: 'CAS源码之DelegatingFilterProxyRegistrationBean(六)' 
date: 2021-10-26
author: 李新
tags:  CAS
---

### (1). 概述
实际上分析了所有的Filter,没有发现和我们的登录什么关系,后面会调整策略,找到分析的入口.
### (2). 看下DelegatingFilterProxyRegistrationBean初始化
```
package org.springframework.boot.autoconfigure.security.servlet;

// **********************************************************************************************
// 这是spring boot对spring security的自动化配置
// **********************************************************************************************
@Configuration(proxyBeanMethods = false)
@ConditionalOnWebApplication(type = Type.SERVLET)
@EnableConfigurationProperties(SecurityProperties.class)
@ConditionalOnClass({ AbstractSecurityWebApplicationInitializer.class, SessionCreationPolicy.class })
@AutoConfigureAfter(SecurityAutoConfiguration.class)
public class SecurityFilterAutoConfiguration {
	
	// **********************************************************************************************
	// springSecurityFilterChain
	// **********************************************************************************************
	private static final String DEFAULT_FILTER_NAME = AbstractSecurityWebApplicationInitializer.DEFAULT_FILTER_NAME;
	
	// **********************************************************************************************
	// DelegatingFilterProxyRegistrationBean的作用是:创建一个Filter,它的名称为:springSecurityFilterChain
	// 向spring中注册bean(securityFilterChainRegistration = DelegatingFilterProxyRegistrationBean)
	// **********************************************************************************************
	@Bean
	@ConditionalOnBean(name = DEFAULT_FILTER_NAME)
	public DelegatingFilterProxyRegistrationBean securityFilterChainRegistration(SecurityProperties securityProperties) {
		// **********************************************************************************************
		// 注册一个名称为:springSecurityFilterChain的Filter(DelegatingFilterProxy)
		// **********************************************************************************************
		DelegatingFilterProxyRegistrationBean registration = new DelegatingFilterProxyRegistrationBean(DEFAULT_FILTER_NAME);
		registration.setOrder(securityProperties.getFilter().getOrder());
		registration.setDispatcherTypes(getDispatcherTypes(securityProperties));
		return registration;
	} // end securityFilterChainRegistration
}
```
### (3). DelegatingFilterProxyRegistrationBean
```
// 1. 继承于父类:AbstractFilterRegistrationBean
public class DelegatingFilterProxyRegistrationBean extends AbstractFilterRegistrationBean<DelegatingFilterProxy> implements ApplicationContextAware {
   
   // 最终的Filter(DelegatingFilterProxy)
   @Override
   	public DelegatingFilterProxy getFilter() {
   		return new DelegatingFilterProxy(this.targetBeanName, getWebApplicationContext()) {
   			@Override
   			protected void initFilterBean() throws ServletException {
   				// Don't initialize filter bean on init()
   			}
   
   		};
   	}
   
} // end DelegatingFilterProxyRegistrationBean


// AbstractFilterRegistrationBean
public abstract class AbstractFilterRegistrationBean<T extends Filter> extends DynamicRegistrationBean<Dynamic> {
	@Override
	protected Dynamic addRegistration(String description, ServletContext servletContext) {
		Filter filter = getFilter();
		return servletContext.addFilter(getOrDeduceName(filter), filter);
	} // end addRegistration
} // end AbstractFilterRegistrationBean
```
### (4). DelegatingFilterProxy
```
// DelegatingFilterProxy属于Filter的实现类,所以,只要关心:doFilter方法即可.

public class DelegatingFilterProxy extends GenericFilterBean {
	public void doFilter(ServletRequest request, ServletResponse response, FilterChain filterChain) throws ServletException, IOException {
		Filter delegateToUse = this.delegate;
		if (delegateToUse == null) {
			synchronized (this.delegateMonitor) {
				delegateToUse = this.delegate;
				if (delegateToUse == null) {
					WebApplicationContext wac = findWebApplicationContext();
					if (wac == null) {
						throw new IllegalStateException("No WebApplicationContext found: " + "no ContextLoaderListener or DispatcherServlet registered?");
					}
					
					// **********************************************************************************
					// 从spring容器中,获得bean名称为:springSecurityFilterChain,并且,类型是:Filter的实现类
					// **********************************************************************************
					delegateToUse = initDelegate(wac);
				}
				this.delegate = delegateToUse;
			}
		}

		// Let the delegate perform the actual doFilter operation.
		invokeDelegate(delegateToUse, request, response, filterChain);
	}// end doFilter
	
	
	protected Filter initDelegate(WebApplicationContext wac) throws ServletException {
		// springSecurityFilterChain
		String targetBeanName = getTargetBeanName();
		
		Assert.state(targetBeanName != null, "No target bean name set");
		// 从Spring容器中获得:springSecurityFilterChain
		Filter delegate = wac.getBean(targetBeanName, Filter.class);
		if (isTargetFilterLifecycle()) {
			delegate.init(getFilterConfig());
		}
		return delegate;
	} // end initDelegate
}	
```
### (4). springSecurityFilterChain何时初始化的?
> springSecurityFilterChain这个bean的初始化在:WebSecurityConfiguration(这个类是:Spring Security里的)类里

```
package org.springframework.security.config.annotation.web.configuration;

@Configuration(proxyBeanMethods = false)
public class WebSecurityConfiguration implements ImportAware, BeanClassLoaderAware {
	
	// ***********************************************************************************************
	// AbstractSecurityWebApplicationInitializer.DEFAULT_FILTER_NAME = springSecurityFilterChain
	// springSecurityFilterChain是在这里创建的
	// ***********************************************************************************************
	@Bean(name = AbstractSecurityWebApplicationInitializer.DEFAULT_FILTER_NAME)
	public Filter springSecurityFilterChain() throws Exception {
		boolean hasConfigurers = this.webSecurityConfigurers != null && !this.webSecurityConfigurers.isEmpty();
		boolean hasFilterChain = !this.securityFilterChains.isEmpty();
		
		Assert.state(!(hasConfigurers && hasFilterChain), "Found WebSecurityConfigurerAdapter as well as SecurityFilterChain. Please select just one.");
		if (!hasConfigurers && !hasFilterChain) {
			WebSecurityConfigurerAdapter adapter = this.objectObjectPostProcessor .postProcess(new WebSecurityConfigurerAdapter() {});
			this.webSecurity.apply(adapter);
		}
		
		for (SecurityFilterChain securityFilterChain : this.securityFilterChains) {
			this.webSecurity.addSecurityFilterChainBuilder(() -> securityFilterChain);
			for (Filter filter : securityFilterChain.getFilters()) {
				if (filter instanceof FilterSecurityInterceptor) {
					this.webSecurity.securityInterceptor((FilterSecurityInterceptor) filter);
					break;
				}
			}
		}
		
		for (WebSecurityCustomizer customizer : this.webSecurityCustomizers) {
			customizer.customize(this.webSecurity);
		}
		
		// Filter == FilterChainProxy
		return this.webSecurity.build();
	} // end 
	
}	
```
### (5). 总结
DelegatingFilterProxyRegistrationBean的职责如下(好像还是没有分析到我们要的核心): 
+ 向Spring中注册一个Filter,Filter的名称为:springSecurityFilterChain(xml中名称的定义而已,并非真正的bean名称),而实现类为:DelegatingFilterProxy   
+ DelegatingFilterProxy属于Filter的实现类,自然会实现:doFilter方法,而该方法又会从Spring容器中找到spring中名称为:springSecurityFilterChain,类型为:Filter的实现类(FilterChainProxy).  
+ FilterChainProxy属于Spring Secuiryt的内容,会对请求进行一些过滤(比如:黑白名单).  
