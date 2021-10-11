---
layout: post
title: 'Spring Security源码之DelegatingFilterProxy(四)' 
date: 2021-10-11
author: 李新
tags:  SpringSecurity
---

### (1). 概述
通过前面的源码分析,能得出一个结论:Spring Security会向Spring注册一个Filter(DelegatingFilterProxy),在这一小节,剖析:DelegatingFilterProxy的初始化过程.   

### (2). 看下DelegatingFilterProxy类关系图
> 从类的关系图上能得出以下几个结论(从结论来看,可以分成两步来分析,分析启动时的初始化,然后,再分析http请求):   
+ 实现了InitializingBean接口,所以,在容器初始化时,会回调:afterPropertiesSet.   
+ 实现了Filter接口,所以,在发起http请求时,会触发:doFilter方法(注意:这个优先级可比Spring MVC的DispatcherServlet还要早).   

!["DelegatingFilterProxy类继承关系图"](/assets/spring-security/imgs/DelegationgFilterProxy.jpg)   

### (3). DelegatingFilterProxy.doFilter
```
@Nullable
// FilterChainProxy
private volatile Filter delegate;

public void doFilter(ServletRequest request, ServletResponse response, FilterChain filterChain) throws ServletException, IOException {
	Filter delegateToUse = this.delegate;
	if (delegateToUse == null) { // true
		synchronized (this.delegateMonitor) { // 并发,控制代码块,同一时间,只有一个线程能运行.  
			delegateToUse = this.delegate;
			if (delegateToUse == null) {
				
				// 获取ApplicationContext
				WebApplicationContext wac = findWebApplicationContext();
				
				// 获取不到的情况下,抛出异常
				if (wac == null) {
					throw new IllegalStateException("No WebApplicationContext found: " + "no ContextLoaderListener or DispatcherServlet registered?");
				}
				
				// ****************************************************************************
				// 从Spring容器中,获得Bean名称为:springSecurityFilterChain的Filter
				// ****************************************************************************
				delegateToUse = initDelegate(wac);
			}
			this.delegate = delegateToUse;
		}
	}

    // *********************************************************************************
	// 委派给:FilterChainProxy进行处理.
	// *********************************************************************************
	// Let the delegate perform the actual doFilter operation.
	invokeDelegate(delegateToUse, request, response, filterChain);
}
```
### (4). DelegatingFilterProxy.initDelegate
```
// 从Spring容器中,获得bean名称(springSecurityFilterChain)的Bean出来
protected Filter initDelegate(WebApplicationContext wac) throws ServletException {
	// targetBeanName =  springSecurityFilterChain
	String targetBeanName = getTargetBeanName();
	Assert.state(targetBeanName != null, "No target bean name set");
	
	// 从Spring容器中获得
	Filter delegate = wac.getBean(targetBeanName, Filter.class);
	if (isTargetFilterLifecycle()) {
		// 调用:FilterChainProxy.init方法,传递配置文件
		delegate.init(getFilterConfig());
	}
	
	return delegate;
}
```
### (5). DelegatingFilterProxy.invokeDelegate
```
// delegate = FilterChainProxy
protected void invokeDelegate(Filter delegate, ServletRequest request, ServletResponse response, FilterChain filterChain) throws ServletException, IOException {
	delegate.doFilter(request, response, filterChain);
}
```
### (6). 总结
DelegatingFilterProxy从Spring容器中拿出名称为:springSecurityFilterChain的对象(FilterChainProxy),最终:会把请求委派给了FilterChainProxy进行处理.   
