---
layout: post
title: 'Spring Security源码之SecurityConfigurer(五)' 
date: 2021-10-11
author: 李新
tags:  SpringSecurity
---

### (1). 概述
在大多数的情况下,我们会继承WebSecurityConfigurerAdapter,然后,重写configure,进行一些我们自定义的配置(HttpSecurity),在这里,首先重点关注:SecurityConfigurer,因为:这就是我们要配置的模型.    

### (2). 看下SecurityConfigurer有哪些实现类
> 从图中有看出来,Spring Security让所有的模块(cqrs/oauth等)配置,都实现了:SecurityConfigurer接口,那么问题来了,HttpSecurity和SecurityConfigurer接口是什么关系呢?      

!["SecurityConfigurer"](/assets/spring-security/imgs/SecurityConfigurer.png)   

```
org.springframework.security.config.annotation.SecurityConfigurer

	org.springframework.security.config.annotation.SecurityConfigurerAdapter

		org.springframework.security.config.annotation.web.configurers.AbstractHttpConfigurer
			org.springframework.security.config.annotation.web.configurers.HttpBasicConfigurer
			org.springframework.security.config.annotation.web.configurers.LogoutConfigurer
			org.springframework.security.config.annotation.web.configurers.RememberMeConfigurer
			org.springframework.security.config.annotation.web.configurers.RequestCacheConfigurer
			org.springframework.security.config.annotation.web.configurers.ServletApiConfigurer
			org.springframework.security.config.annotation.web.configurers.DefaultLoginPageConfigurer
			org.springframework.security.config.annotation.web.configurers.SessionManagementConfigurer
			org.springframework.security.config.annotation.web.configurers.PortMapperConfigurer
			org.springframework.security.config.annotation.web.configurers.ExceptionHandlingConfigurer
			org.springframework.security.config.annotation.web.configurers.HeadersConfigurer
			org.springframework.security.config.annotation.web.configurers.CsrfConfigurer
			org.springframework.security.config.annotation.web.configurers.oauth2.client.ImplicitGrantConfigurer
			org.springframework.security.config.annotation.web.configurers.JeeConfigurer
			org.springframework.security.config.annotation.web.configurers.AnonymousConfigurer
			org.springframework.security.config.annotation.web.configurers.ChannelSecurityConfigurer
			org.springframework.security.config.annotation.web.configurers.CorsConfigurer
			org.springframework.security.config.annotation.web.configurers.SecurityContextConfigurer
			org.springframework.security.config.annotation.web.configurers.X509Configurer

		org.springframework.security.config.annotation.web.configurers.AbstractAuthenticationFilterConfigurer
			org.springframework.security.config.annotation.web.configurers.FormLoginConfigurer
			org.springframework.security.config.annotation.web.configurers.oauth2.client.OAuth2LoginConfigurer
			org.springframework.security.config.annotation.web.configurers.openid.OpenIDLoginConfigurer

		org.springframework.security.config.annotation.web.configurers.AbstractInterceptUrlConfigurer
			org.springframework.security.config.annotation.web.configurers.UrlAuthorizationConfigurer
			org.springframework.security.config.annotation.web.configurers.ExpressionUrlAuthorizationConfigurer
			
		org.springframework.security.config.annotation.authentication.configurers.ldap.LdapAuthenticationProviderConfigurer
```
### (3). 查看HttpSecurity类图
> 从UML图中,能看出HttpSecurity内部包含着一个Map集合,这个集合要求是SecurityConfigurer类型的,也就是说通过HttpSecurity的所有操作,实际是把信息载体,设置在集合(Map)里.

!["HttpSecurity"](/assets/spring-security/imgs/HttpSecurity.jpg)

### (4). HttpSecurity
```
public final class HttpSecurity extends
		AbstractConfiguredSecurityBuilder<DefaultSecurityFilterChain, HttpSecurity>
		implements SecurityBuilder<DefaultSecurityFilterChain>,
		HttpSecurityBuilder<HttpSecurity> {

    
	public OpenIDLoginConfigurer<HttpSecurity> openidLogin() throws Exception {
		return getOrApply(new OpenIDLoginConfigurer<>());
	}
	
	public HeadersConfigurer<HttpSecurity> headers() throws Exception {
		return getOrApply(new HeadersConfigurer<>());
	}
	
	public CorsConfigurer<HttpSecurity> cors() throws Exception {
		return getOrApply(new CorsConfigurer<>());
	}
	
	public SessionManagementConfigurer<HttpSecurity> sessionManagement() throws Exception {
		return getOrApply(new SessionManagementConfigurer<>());
	}
	
	public PortMapperConfigurer<HttpSecurity> portMapper() throws Exception {
		return getOrApply(new PortMapperConfigurer<>());
	}
	
	public JeeConfigurer<HttpSecurity> jee() throws Exception {
		return getOrApply(new JeeConfigurer<>());
	}
	
	public X509Configurer<HttpSecurity> x509() throws Exception {
		return getOrApply(new X509Configurer<>());
	}
	
	public RememberMeConfigurer<HttpSecurity> rememberMe() throws Exception {
		return getOrApply(new RememberMeConfigurer<>());
	}
	
	public ExpressionUrlAuthorizationConfigurer<HttpSecurity>.ExpressionInterceptUrlRegistry authorizeRequests()
				throws Exception {
		ApplicationContext context = getContext();
		return getOrApply(new ExpressionUrlAuthorizationConfigurer<>(context))
				.getRegistry();
	}
	
	public RequestCacheConfigurer<HttpSecurity> requestCache() throws Exception {
		return getOrApply(new RequestCacheConfigurer<>());
	}
	
	public ExceptionHandlingConfigurer<HttpSecurity> exceptionHandling() throws Exception {
		return getOrApply(new ExceptionHandlingConfigurer<>());
	}
	
	public SecurityContextConfigurer<HttpSecurity> securityContext() throws Exception {
		return getOrApply(new SecurityContextConfigurer<>());
	}
	
	public ServletApiConfigurer<HttpSecurity> servletApi() throws Exception {
		return getOrApply(new ServletApiConfigurer<>());
	}
	
	public CsrfConfigurer<HttpSecurity> csrf() throws Exception {
		ApplicationContext context = getContext();
		return getOrApply(new CsrfConfigurer<>(context));
	}
	
	public LogoutConfigurer<HttpSecurity> logout() throws Exception {
		return getOrApply(new LogoutConfigurer<>());
	}
	
	public AnonymousConfigurer<HttpSecurity> anonymous() throws Exception {
		return getOrApply(new AnonymousConfigurer<>());
	}
	
	public FormLoginConfigurer<HttpSecurity> formLogin() throws Exception {
		return getOrApply(new FormLoginConfigurer<>());
	}
	
	public OAuth2LoginConfigurer<HttpSecurity> oauth2Login() throws Exception {
		return getOrApply(new OAuth2LoginConfigurer<>());
	}
	
	public ChannelSecurityConfigurer<HttpSecurity>.ChannelRequestMatcherRegistry requiresChannel()
				throws Exception {
		ApplicationContext context = getContext();
		return getOrApply(new ChannelSecurityConfigurer<>(context))
				.getRegistry();
	}
	
	public HttpBasicConfigurer<HttpSecurity> httpBasic() throws Exception {
		return getOrApply(new HttpBasicConfigurer<>());
	}
	
	public RequestMatcherConfigurer requestMatchers() {
		return requestMatcherConfigurer;
	}
	
	public HttpSecurity requestMatcher(RequestMatcher requestMatcher) {
		this.requestMatcher = requestMatcher;
		return this;
	}
	
	public HttpSecurity antMatcher(String antPattern) {
		return requestMatcher(new AntPathRequestMatcher(antPattern));
	}
	
	public HttpSecurity mvcMatcher(String mvcPattern) {
		HandlerMappingIntrospector introspector = new HandlerMappingIntrospector(getContext());
		return requestMatcher(new MvcRequestMatcher(introspector, mvcPattern));
	}
	
	public HttpSecurity regexMatcher(String pattern) {
		return requestMatcher(new RegexRequestMatcher(pattern, null));
	}
}
```
### (5). 总结
在HttpSecurity的内部,持有一个Map集合,包含着所有的SecurityConfigurer的实现类,当我们进行配置时,实际是把这些配置的信息,保存在Map里,以供使用.   