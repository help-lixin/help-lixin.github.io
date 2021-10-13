---
layout: post
title: 'Spring Security源码之WebSecurity(三)' 
date: 2021-10-11
author: 李新
tags:  SpringSecurity
---

### (1). 概述
在前面,只剖析了创建:springSecurityFilterChain(Filter)的入口,并未,深入去剖析springSecurityFilterChain是如何创建出来的,在这小节,主要剖析这个功能.   

### (2). 看下WebSecurity的类结构
!["WebSecurity"](/assets/spring-security/imgs/WebSecurity.jpg)

### (3). AbstractSecurityBuilder
```

private AtomicBoolean building = new AtomicBoolean();
private O object;

public final O build() throws Exception {
	if (this.building.compareAndSet(false, true)) {  // 元子性控制,这个代码块只执行一次.
		// ***************************************************************************
		// 委托给子类:AbstractConfiguredSecurityBuilder.doBuild
		// ***************************************************************************
		this.object = doBuild();
		
		return this.object;
	}
	throw new AlreadyBuiltException("This object has already been built");
}
```
### (4). AbstractConfiguredSecurityBuilder
```
private final LinkedHashMap<Class<? extends SecurityConfigurer<O, B>>, List<SecurityConfigurer<O, B>>> configurers = new LinkedHashMap<Class<? extends SecurityConfigurer<O, B>>, List<SecurityConfigurer<O, B>>>();
private final List<SecurityConfigurer<O, B>> configurersAddedInInitializing = new ArrayList<SecurityConfigurer<O, B>>();

protected final O doBuild() throws Exception {
	synchronized (configurers) {
		buildState = BuildState.INITIALIZING;

		beforeInit();
		// ***************************************************
		// 1. 委派给init方法.
		// ***************************************************
		init();

		buildState = BuildState.CONFIGURING;

		beforeConfigure();
		configure();

		buildState = BuildState.BUILDING;
		
		// ******************************************************************************
		// 委托给子类:WebSecurity.performBuild
		// ******************************************************************************
		O result = performBuild();

		buildState = BuildState.BUILT;

		return result;
	}
} // end 


// 2. 获得所有的:SecurityConfigurer
private void init() throws Exception {
	Collection<SecurityConfigurer<O, B>> configurers = getConfigurers();
    
	// 遍历所有的:SecurityConfigurer,并调用init方法.
	for (SecurityConfigurer<O, B> configurer : configurers) {
		configurer.init((B) this);
	}
	
	for (SecurityConfigurer<O, B> configurer : configurersAddedInInitializing) {
		configurer.init((B) this);
	}
}

```
### (5). WebSecurity
```

// 在鉴权时,要忽略的URL,会被包装成:DefaultSecurityFilterChain
private final List<RequestMatcher> ignoredRequests = new ArrayList<>();
// SecurityFilterChain
private final List<SecurityBuilder<? extends SecurityFilterChain>> securityFilterChainBuilders = new ArrayList<SecurityBuilder<? extends SecurityFilterChain>>();


protected Filter performBuild() throws Exception {
	Assert.state(!securityFilterChainBuilders.isEmpty(),"At least one SecurityBuilder<? extends SecurityFilterChain> needs to be specified. Typically this done by adding a @Configuration that extends WebSecurityConfigurerAdapter. More advanced users can invoke " + WebSecurity.class.getSimpleName() + ".addSecurityFilterChainBuilder directly");
	
	int chainSize = ignoredRequests.size() + securityFilterChainBuilders.size();
	List<SecurityFilterChain> securityFilterChains = new ArrayList<>(chainSize);
	
	
	for (RequestMatcher ignoredRequest : ignoredRequests) {
		securityFilterChains.add(new DefaultSecurityFilterChain(ignoredRequest));
	}
	
	for (SecurityBuilder<? extends SecurityFilterChain> securityFilterChainBuilder : securityFilterChainBuilders) {
		securityFilterChains.add(securityFilterChainBuilder.build());
	}
	
	// **************************************************************************************
	// 通过FilterChainProxy,包裹着所有的:SecurityFilterChain
	// **************************************************************************************
	FilterChainProxy filterChainProxy = new FilterChainProxy(securityFilterChains);
	
	if (httpFirewall != null) {
		filterChainProxy.setFirewall(httpFirewall);
	}
	// 没有交给Spring,而是自己初始化了一把
	filterChainProxy.afterPropertiesSet();
	
	Filter result = filterChainProxy;
	if (debugEnabled) {
		logger.warn("\n\n"
				+ "********************************************************************\n"
				+ "**********        Security debugging is enabled.       *************\n"
				+ "**********    This may include sensitive information.  *************\n"
				+ "**********      Do not use in a production system!     *************\n"
				+ "********************************************************************\n\n");
		result = new DebugFilter(filterChainProxy);
	}
	postBuildAction.run();
	return result;
}
```
### (6). FilterChainProxy包含了哪些Filter呢?
```
# 以下为:DefaultSecurityFilterChain的所有Filter
filterChains = org.springframework.security.web.DefaultSecurityFilterChain = 
[ 
	requestMatcher = org.springframework.security.web.util.matcher.AnyRequestMatcher@1, 
	filters = [
		org.springframework.security.web.context.request.async.WebAsyncManagerIntegrationFilter@3f018494, 
		org.springframework.security.web.context.SecurityContextPersistenceFilter@732f6050, 
		org.springframework.security.web.header.HeaderWriterFilter@2ddb3ae8, 
		org.springframework.security.web.csrf.CsrfFilter@d2291de, 
		org.springframework.security.web.authentication.logout.LogoutFilter@7cca01a8, 
		org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter@4db60246, 
		org.springframework.security.web.authentication.ui.DefaultLoginPageGeneratingFilter@33feb805, 
		org.springframework.security.web.authentication.www.BasicAuthenticationFilter@eb6ec6, 
		org.springframework.security.web.savedrequest.RequestCacheAwareFilter@30c4e352, 
		org.springframework.security.web.servletapi.SecurityContextHolderAwareRequestFilter@5f14761c, 
		org.springframework.security.web.authentication.AnonymousAuthenticationFilter@3ce443f9, 
		org.springframework.security.web.session.SessionManagementFilter@3c91530d, 
		org.springframework.security.web.access.ExceptionTranslationFilter@17ba57f0, 
		org.springframework.security.web.access.intercept.FilterSecurityInterceptor@66f28a1f
	]
 ]
```
### (7). 总结
WebSecurity.build方法,最终会创建:FilterChainProxy对象,而FilterChainProxy是一个责任链条来着的.   