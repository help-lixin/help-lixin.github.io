---
layout: post
title: 'Spring Security源码之FilterSecurityInterceptor(十)' 
date: 2021-10-11
author: 李新
tags:  SpringSecurity
---

### (1). 概述
前面对认证部份的源码进行了剖析,在这里将对,鉴权部份进行剖析,那么,如何找到鉴权的入口呢?带着这个疑问,我们来看源码.

### (2). WebSecurityConfigurerAdapter
> 在前面的分析,我们知道,HttpSecurity内部持有SecurityConfigurer集合,HttpSecurity会对这个集合进行初始化(init/configure).

```
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
		// ***********************************************************************************************************
		// 通过:HttpSecurity配置权限
		// ***********************************************************************************************************
        ExpressionUrlAuthorizationConfigurer<HttpSecurity>.ExpressionInterceptUrlRegistry registry = http.authorizeRequests();
		registry
				// 允许/login.html不需要认证.
				.antMatchers("/login.html").permitAll()
				// ***************************************************************
				// 配置hello请求,需要具备有ADMIN角色,才能放行.
				// ***************************************************************
				.antMatchers("/hello").hasRole("ADMIN")
				
				// 所有请求都要登录之后,才能访问
				.anyRequest().authenticated();
    } // end configure
}		
```
### (3). ExpressionUrlAuthorizationConfigurer
> 从代码中,能分析出来,整个鉴权是交给了:FilterSecurityInterceptor,在FilterSecurityInterceptor内部又持有:AuthenticationManager/AccessDecisionManager.   

```
// ExpressionUrlAuthorizationConfigurer 继承于 AbstractInterceptUrlConfigurer
public final class ExpressionUrlAuthorizationConfigurer<H extends HttpSecurityBuilder<H>>
		extends AbstractInterceptUrlConfigurer<ExpressionUrlAuthorizationConfigurer<H>, H> {
			
} // end ExpressionUrlAuthorizationConfigurer


abstract class AbstractInterceptUrlConfigurer<C extends AbstractInterceptUrlConfigurer<C, H>, H extends HttpSecurityBuilder<H>>
		extends AbstractHttpConfigurer<C, H> {
    
	
	@Override
	public void configure(H http) throws Exception {
		FilterInvocationSecurityMetadataSource metadataSource = createMetadataSource(http);
		if (metadataSource == null) {
			return;
		}
		
		// ************************************************************************************************
		// 1.创建:FilterSecurityInterceptor
		// ************************************************************************************************
		FilterSecurityInterceptor securityInterceptor = createFilterSecurityInterceptor(http, metadataSource, http.getSharedObject(AuthenticationManager.class));
		if (filterSecurityInterceptorOncePerRequest != null) {
			securityInterceptor.setObserveOncePerRequest(filterSecurityInterceptorOncePerRequest);
		}
		
		securityInterceptor = postProcess(securityInterceptor);
		// 
		http.addFilter(securityInterceptor);
		http.setSharedObject(FilterSecurityInterceptor.class, securityInterceptor);
	} // end configure
	
	
	private FilterSecurityInterceptor createFilterSecurityInterceptor(H http, FilterInvocationSecurityMetadataSource metadataSource, AuthenticationManager authenticationManager) throws Exception {
		FilterSecurityInterceptor securityInterceptor = new FilterSecurityInterceptor();
		securityInterceptor.setSecurityMetadataSource(metadataSource);
		// 配置AuthenticationManager/AccessDecisionManager
		securityInterceptor.setAccessDecisionManager(getAccessDecisionManager(http));
		securityInterceptor.setAuthenticationManager(authenticationManager);
		securityInterceptor.afterPropertiesSet();
		return securityInterceptor;
	} // end createFilterSecurityInterceptor
	
} // end AbstractInterceptUrlConfigurer
```
### (4). 查看FilterSecurityInterceptor的类关系
```
org.springframework.security.access.intercept.AbstractSecurityInterceptor
	org.springframework.security.web.access.intercept.FilterSecurityInterceptor
```
### (5). FilterSecurityInterceptor.doFilter
>  由于FilterSecurityInterceptor实现了Filter,所以,关心doFilter即可

```
public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
	// 1. 直接把ServletRequest/ServletResponse/FilterChain 包装成:FilterInvocation对象.
	FilterInvocation fi = new FilterInvocation(request, response, chain);
	// **********************************************************************************************
	// 2. 调用invoke方法
	// **********************************************************************************************
	invoke(fi);
}
```
### (6). FilterSecurityInterceptor.invoke
```

private static final String FILTER_APPLIED = "__spring_security_filterSecurityInterceptor_filterApplied";
// **********************************************************************************************
// 1. FilterInvocationSecurityMetadataSource有点类似于SpringMVC中的:HandlerMapping,保存着所有的请求映射信息
//     /hello --> hasRole("ADMIN")
// **********************************************************************************************
private FilterInvocationSecurityMetadataSource securityMetadataSource;
private boolean observeOncePerRequest = true;


public void invoke(FilterInvocation fi) throws IOException, ServletException {
	// 
	if ((fi.getRequest() != null) && (fi.getRequest().getAttribute(FILTER_APPLIED) != null) && observeOncePerRequest) {
		// filter already applied to this request and user wants us to observe
		// once-per-request handling, so don't re-do security checking
		fi.getChain().doFilter(fi.getRequest(), fi.getResponse());
	} else {
		// first time this request being called, so perform security checking
		if (fi.getRequest() != null && observeOncePerRequest) {
			fi.getRequest().setAttribute(FILTER_APPLIED, Boolean.TRUE);
		}
		
		// **************************************************************************************
		// 委托给父类(AbstractSecurityInterceptor)
		// **************************************************************************************
		InterceptorStatusToken token = super.beforeInvocation(fi);

		try {
			// 继续向下一个调用链传递调用.
			fi.getChain().doFilter(fi.getRequest(), fi.getResponse());
		} finally {
			// 清场操作
			super.finallyInvocation(token);
		}
		
		super.afterInvocation(token, null);
	}
}
```
### (7). AbstractSecurityInterceptor.beforeInvocation
```
protected InterceptorStatusToken beforeInvocation(Object object) {
	Assert.notNull(object, "Object was null");
	final boolean debug = logger.isDebugEnabled();
	
	
	if (!getSecureObjectClass().isAssignableFrom(object.getClass())) {
		throw new IllegalArgumentException("Security invocation attempted for object " + object.getClass().getName() + " but AbstractSecurityInterceptor only configured to support secure objects of type: " + getSecureObjectClass());
	}

	// *************************************************************************************************
	// 1. 查询发起的请求是否在元数据里
	// SecurityMetadataSource 里记录着所有的URL请求与元信息
	// /login.html --> permitAll
	// /hello      --> hasRole("ADMIN")
	// /**         --> permitAll
	// *************************************************************************************************
	Collection<ConfigAttribute> attributes = this.obtainSecurityMetadataSource().getAttributes(object);
	
	// 相当于请求的URL信息找不到的情况下
	if (attributes == null || attributes.isEmpty()) {
		if (rejectPublicInvocations) { // 抛异常
			throw new IllegalArgumentException("Secure object invocation " + object + " was denied as public invocations are not allowed via this interceptor. " + "This indicates a configuration error because the " + "rejectPublicInvocations property is set to 'true'");
		}

		if (debug) {
			logger.debug("Public object - authentication not attempted");
		}

		// 发布事件
		publishEvent(new PublicInvocationEvent(object));

		return null; // no further work post-invocation
	}

	if (debug) {
		logger.debug("Secure object: " + object + "; Attributes: " + attributes);
	}
	
	
	// 在UsernamePasswordAuthenticationFilter里就已经设置了Authentication,所以,这边能拿到:Authentication
	if (SecurityContextHolder.getContext().getAuthentication() == null) {
		credentialsNotFound(messages.getMessage( "AbstractSecurityInterceptor.authenticationNotFound", "An Authentication object was not found in the SecurityContext"),object, attributes);
	}
	
	// ***************************************************************************
	// 2. 在鉴权之前,必须先有做过认证来着的,如果没有认证,则会调用:Authentication进行认证.
	// ***************************************************************************
	Authentication authenticated = authenticateIfRequired();

	// Attempt authorization
	try {
		// ******************************************************************************************
		// 3. 委托给访问决策管理进行决定(AccessDecisionManager),这里是重点,但是,暂时先不分析,后面我会留一节,专门分析这个接口:AccessDecisionManager
		// ******************************************************************************************
		this.accessDecisionManager.decide(authenticated, object, attributes);
	} catch (AccessDeniedException accessDeniedException) {
		publishEvent(new AuthorizationFailureEvent(object, attributes, authenticated, accessDeniedException));
		throw accessDeniedException;
	}

	if (debug) {
		logger.debug("Authorization successful");
	}
	
	// 发布:AuthorizedEvent,所以,可以无缝嵌入做一些你想做的事情.
	if (publishAuthorizationSuccess) {
		publishEvent(new AuthorizedEvent(object, attributes, authenticated));
	}

	// Attempt to run as a different user
	Authentication runAs = this.runAsManager.buildRunAs(authenticated, object , attributes);

	if (runAs == null) {  // true
		if (debug) {
			logger.debug("RunAsManager did not change Authentication object");
		}
		
		// 通过	InterceptorStatusToken包裹着:Authentication
		// no further work post-invocation
		return new InterceptorStatusToken(SecurityContextHolder.getContext(), false, attributes, object);
	} else { 
		// *****************************************************************************
		// 没太理解这个逻辑是想做什么?
		// *****************************************************************************
		if (debug) {
			logger.debug("Switching to RunAs Authentication: " + runAs);
		}

		SecurityContext origCtx = SecurityContextHolder.getContext();
		SecurityContextHolder.setContext(SecurityContextHolder.createEmptyContext());
		SecurityContextHolder.getContext().setAuthentication(runAs);

		// need to revert to token.Authenticated post-invocation
		return new InterceptorStatusToken(origCtx, true, attributes, object);
	} 
}
```
### (8). AbstractSecurityInterceptor.authenticateIfRequired
```
private Authentication authenticateIfRequired() {
	// 从上下文中获得:Authentication
	Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
     
	// 看下是否已经认证
	if (authentication.isAuthenticated() && !alwaysReauthenticate) {
		if (logger.isDebugEnabled()) {
			logger.debug("Previously Authenticated: " + authentication);
		}
		// 认证过了的情况下,直接返回:Authentication
		return authentication;
	}
     
	//  到了这一步:代表上下文中的Authentication对象,还未被认证,需要调用:AuthenticationManager进行认证,前面有分析过,在这里就不多分析了.
	authentication = authenticationManager.authenticate(authentication);

	// We don't authenticated.setAuthentication(true), because each provider should do
	// that
	if (logger.isDebugEnabled()) {
		logger.debug("Successfully Authenticated: " + authentication);
	}

	// 然后,再把Authentication设置到上下文中.
	SecurityContextHolder.getContext().setAuthentication(authentication);

	return authentication;
}
```
### (9). 总结
从对FilterSecurityInterceptor的源码剖析来看,它的主要职责如下:  
+ 根据当前的请求,从SecurityMetadataSource里,查找URL与元数据的映射.  
+ 验证Authentication是否已经认证过.  
+ 委派给决策管理(AccessDecisionManager),进行决策(鉴权),鉴权失败的情况下会抛出:AccessDeniedException异常.   
