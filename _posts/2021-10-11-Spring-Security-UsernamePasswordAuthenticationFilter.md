---
layout: post
title: 'Spring Security源码之UsernamePasswordAuthenticationFilter(七)' 
date: 2021-10-11
author: 李新
tags:  SpringSecurity
---

### (1). 概述
在这一篇,主要剖析当登录的时候,Spring Security是如可处理的,要剖析的源码就是上一小节,剖析时留下来的Filter(UsernamePasswordAuthenticationFilter)  

### (2). 看下UsernamePasswordAuthenticationFilter关系图
```
org.springframework.web.filter.GenericFilterBean   # GenericFilterBean是Spring对Filter的扩展
	org.springframework.security.web.authentication.AbstractAuthenticationProcessingFilter
		org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter
```
### (3). AbstractAuthenticationProcessingFilter
> 我们知道,对于Filter应该要看doFilter方法,所以,AbstractAuthenticationProcessingFilter是对doFilter的实现,然后,预留一小部份给了:UsernamePasswordAuthenticationFilter

```
public abstract class AbstractAuthenticationProcessingFilter 
				// ***********************************************************************************
				// GenericFilterBean
				// ***********************************************************************************
                extends GenericFilterBean
				implements ApplicationEventPublisherAware, MessageSourceAware {

	// 把用户名和密码,变成:UsernamePasswordAuthenticationToken后,会回调:AuthenticationDetailsSource进行配置:UsernamePasswordAuthenticationToken的相关信息
	protected AuthenticationDetailsSource<HttpServletRequest, ?> authenticationDetailsSource = new WebAuthenticationDetailsSource();
	
	// ********************************************************************************************************
	// 2. 最终要委托给:AuthenticationManager
	// ********************************************************************************************************
	private AuthenticationManager authenticationManager;
	protected MessageSourceAccessor messages = SpringSecurityMessageSource.getAccessor();
	private RememberMeServices rememberMeServices = new NullRememberMeServices();

	private RequestMatcher requiresAuthenticationRequestMatcher;
    
	// 是否跳过成功后的回调函数
	private boolean continueChainBeforeSuccessfulAuthentication = false;

    // ***********************************************************************************************
	// Session管理策略(典型的策略模式)
	// ***********************************************************************************************
	private SessionAuthenticationStrategy sessionStrategy = new NullAuthenticatedSessionStrategy();

	private boolean allowSessionCreation = true;

    // ***************************************************************************************************
	// 登录成功/登录失败相应的Handler
	// ***************************************************************************************************
	private AuthenticationSuccessHandler successHandler = new SavedRequestAwareAuthenticationSuccessHandler();
	private AuthenticationFailureHandler failureHandler = new SimpleUrlAuthenticationFailureHandler();
	
	
	public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
				throws IOException, ServletException {
					
		HttpServletRequest request = (HttpServletRequest) req;
		HttpServletResponse response = (HttpServletResponse) res;
        
		// 如果不是登录(/login)请求,则直接跳过Filter
		if (!requiresAuthentication(request, response)) {
			chain.doFilter(request, response);

			return;
		}
         
		if (logger.isDebugEnabled()) {
			logger.debug("Request is to process authentication");
		}

        // ***********************************************************************************
		// 认证之后的结果.
		// ***********************************************************************************
		Authentication authResult;

		try {
			// ***********************************************************************************
			// 委派给子类(UsernamePasswordAuthenticationFilter)进行处理
			// ***********************************************************************************
			authResult = attemptAuthentication(request, response);
			if (authResult == null) {
				// return immediately as subclass has indicated that it hasn't completed
				// authentication
				return;
			}
			
			// 委托给Session管理策略,在这里我们暂且不管这个
			sessionStrategy.onAuthentication(authResult, request, response);
		} catch (InternalAuthenticationServiceException failed) {
			logger.error("An internal error occurred while trying to authenticate the user.",failed);
			// ***********************************************************************************
			// 认证失败的Handler
			// ***********************************************************************************
			unsuccessfulAuthentication(request, response, failed);
			return;
		} catch (AuthenticationException failed) {
			// ***********************************************************************************
			// 认证失败的Handler
			// ***********************************************************************************
			// Authentication failed
			unsuccessfulAuthentication(request, response, failed);
			return;
		}

        // 可开关配置,是否启用认证成功的Filter
		// Authentication success
		if (continueChainBeforeSuccessfulAuthentication) {
			chain.doFilter(request, response);
		}

		// ***********************************************************************************
		// 认证成功的Handler
		// ***********************************************************************************
		successfulAuthentication(request, response, chain, authResult);
	} // end doFilter
	
	protected void successfulAuthentication(HttpServletRequest request,
				HttpServletResponse response, FilterChain chain, Authentication authResult)
				throws IOException, ServletException {
	
		if (logger.isDebugEnabled()) {
			logger.debug("Authentication success. Updating SecurityContextHolder to contain: "
					+ authResult);
		}

		// ***********************************************************************************
		// 认证成功之后,把Authentication与线程上下文进行绑定.
		// ***********************************************************************************
		SecurityContextHolder.getContext().setAuthentication(authResult);
		
		// ***********************************************************************************
		// 调用记住我功能
		// ***********************************************************************************
		rememberMeServices.loginSuccess(request, response, authResult);

		// Fire event
		if (this.eventPublisher != null) {
			eventPublisher.publishEvent(new InteractiveAuthenticationSuccessEvent(
					authResult, this.getClass()));
		}

		successHandler.onAuthenticationSuccess(request, response, authResult);
	} // end successfulAuthentication
	
}
```
### (4). UsernamePasswordAuthenticationFilter
```
public class UsernamePasswordAuthenticationFilter extends AbstractAuthenticationProcessingFilter {
    public static final String SPRING_SECURITY_FORM_USERNAME_KEY = "username";
	public static final String SPRING_SECURITY_FORM_PASSWORD_KEY = "password";

	private String usernameParameter = SPRING_SECURITY_FORM_USERNAME_KEY;
	private String passwordParameter = SPRING_SECURITY_FORM_PASSWORD_KEY;
	private boolean postOnly = true;
    
	
	public UsernamePasswordAuthenticationFilter() {
		// 设置登录的URLMatcher
		super(new AntPathRequestMatcher("/login", "POST"));
	}
	
	public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException {
		
		// 如果,不是POST请求,则抛出异常(AuthenticationServiceException)
		if (postOnly && !request.getMethod().equals("POST")) {
			throw new AuthenticationServiceException("Authentication method not supported: " + request.getMethod());
		}
		
		// 从请求中获得用户名和密码
		String username = obtainUsername(request);
		String password = obtainPassword(request);

		if (username == null) {
			username = "";
		}

		if (password == null) {
			password = "";
		}

		username = username.trim();

		// ********************************************************************************************
		// 产生Authentication(UsernamePasswordAuthenticationToken)
		// ********************************************************************************************
		UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(username, password);

		// ********************************************************************************************
		// 调用父类,设置:Authentication的一些相关信息
		// ********************************************************************************************
		// Allow subclasses to set the "details" property
		setDetails(request, authRequest);

		// ********************************************************************************************
		// 委托给:AuthenticationManager进行认证管理
		// ********************************************************************************************
		return this.getAuthenticationManager().authenticate(authRequest);
	} // end attemptAuthentication
}
```
### (5). 总结
UsernamePasswordAuthenticationFilter的主要职责就是把登录表单的信息,封装成:Authentication,并委派给:AuthenticationManager进行处理,最后把认证成功后的对象:Authentication与线程上下文进行绑定.      