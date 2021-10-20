---
layout: post
title: 'Spring Security与Oauth2整合源码之OAuth2AuthenticationManager(二)' 
date: 2021-10-11
author: 李新
tags:  SpringSecurity Oauth2
---

### (1). 概述
在这一小篇,主要剖析下:Oauth2与Spring Security结合时,是如何进行认证的.   

### (2). 切入点在哪?
```
@Configuration
// 1. 开启注解(@EnableResourceServer)
@EnableResourceServer
// 2. 切入点就在于我们继承的:ResourceServerConfigurerAdapter
public class ResourceConfig extends ResourceServerConfigurerAdapter {

    @Override
    public void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .anyRequest().authenticated()
                .and()
                .requestMatchers().antMatchers("/user/**");
    }
}
```
### (3). ResourceServerSecurityConfigurer
```
public final class ResourceServerSecurityConfigurer extends
		SecurityConfigurerAdapter<DefaultSecurityFilterChain, HttpSecurity> {

	@Override
	public void configure(HttpSecurity http) throws Exception {

		// **************************************************************************************
		//  创建了一个:AuthenticationManager
		// **************************************************************************************
		AuthenticationManager oauthAuthenticationManager = oauthAuthenticationManager(http);
		
		// **************************************************************************************
		// 创建了一个:Filter(OAuth2AuthenticationProcessingFilter)
		// **************************************************************************************
		resourcesServerFilter = new OAuth2AuthenticationProcessingFilter();
		resourcesServerFilter.setAuthenticationEntryPoint(authenticationEntryPoint);
		resourcesServerFilter.setAuthenticationManager(oauthAuthenticationManager);
		if (eventPublisher != null) {
			resourcesServerFilter.setAuthenticationEventPublisher(eventPublisher);
		}
		if (tokenExtractor != null) {
			resourcesServerFilter.setTokenExtractor(tokenExtractor);
		}
		resourcesServerFilter = postProcess(resourcesServerFilter);
		resourcesServerFilter.setStateless(stateless);

		// @formatter:off
		http
			.authorizeRequests().expressionHandler(expressionHandler)
		.and()
			.addFilterBefore(resourcesServerFilter, AbstractPreAuthenticatedProcessingFilter.class)
			.exceptionHandling()
				.accessDeniedHandler(accessDeniedHandler)
				.authenticationEntryPoint(authenticationEntryPoint);
		// @formatter:on
	}

}
```
### (4). OAuth2AuthenticationManager
```
public class OAuth2AuthenticationManager implements AuthenticationManager, InitializingBean {

	private ResourceServerTokenServices tokenServices;

	private ClientDetailsService clientDetailsService;

	private String resourceId;

	public void setResourceId(String resourceId) {
		this.resourceId = resourceId;
	}

	public void setClientDetailsService(ClientDetailsService clientDetailsService) {
		this.clientDetailsService = clientDetailsService;
	}

	public void setTokenServices(ResourceServerTokenServices tokenServices) {
		this.tokenServices = tokenServices;
	}

	public void afterPropertiesSet() {
		Assert.state(tokenServices != null, "TokenServices are required");
	}

	// **********************************************************************************
	// 认证管理
	// **********************************************************************************
	public Authentication authenticate(Authentication authentication) throws AuthenticationException {
		if (authentication == null) {
			throw new InvalidTokenException("Invalid token (token not found)");
		}
		
		// 1. 获取token
		String token = (String) authentication.getPrincipal();
		
		// 2. 根据token,从ResourceServerTokenServices中加载:OAuth2Authentication
		OAuth2Authentication auth = tokenServices.loadAuthentication(token);
		if (auth == null) {
			throw new InvalidTokenException("Invalid token: " + token);
		}

        // 3. 获得token对应的resourceid
		Collection<String> resourceIds = auth.getOAuth2Request().getResourceIds();
		if (resourceId != null && resourceIds != null && !resourceIds.isEmpty() && !resourceIds.contains(resourceId)) {
			throw new OAuth2AccessDeniedException("Invalid token does not contain resource id (" + resourceId + ")");
		}

		// 4. 检查下scope
		checkClientDetails(auth);
		
		if (authentication.getDetails() instanceof OAuth2AuthenticationDetails) {
			OAuth2AuthenticationDetails details = (OAuth2AuthenticationDetails) authentication.getDetails();
			// Guard against a cached copy of the same details
			if (!details.equals(auth.getDetails())) {
				// Preserve the authentication details from the one loaded by token services
				details.setDecodedDetails(auth.getDetails());
			}
		}
		
		auth.setDetails(authentication.getDetails());
		auth.setAuthenticated(true);
		return auth;

	}

	private void checkClientDetails(OAuth2Authentication auth) {
		if (clientDetailsService != null) {
			ClientDetails client;
			try {
				client = clientDetailsService.loadClientByClientId(auth.getOAuth2Request().getClientId());
			}
			catch (ClientRegistrationException e) {
				throw new OAuth2AccessDeniedException("Invalid token contains invalid client id");
			}
			
			Set<String> allowed = client.getScope();
			for (String scope : auth.getOAuth2Request().getScope()) {
				if (!allowed.contains(scope)) {
					throw new OAuth2AccessDeniedException(
							"Invalid token contains disallowed scope (" + scope + ") for this client");
				}
			}
		}
	}

}
```
### (5). OAuth2AuthenticationProcessingFilter
```
public class OAuth2AuthenticationProcessingFilter implements Filter, InitializingBean {
	
	private AuthenticationEntryPoint authenticationEntryPoint = new OAuth2AuthenticationEntryPoint();
	
	private AuthenticationManager authenticationManager;

	private AuthenticationDetailsSource<HttpServletRequest, ?> authenticationDetailsSource = new OAuth2AuthenticationDetailsSource();

	private TokenExtractor tokenExtractor = new BearerTokenExtractor();

	private AuthenticationEventPublisher eventPublisher = new NullEventPublisher();

	private boolean stateless = true;

	public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException {
		final boolean debug = logger.isDebugEnabled();
		final HttpServletRequest request = (HttpServletRequest) req;
		final HttpServletResponse response = (HttpServletResponse) res;

		try {
			
			// 1. 从http的请求头里获得:Authorization(或请求的参数中获得:access_token),并取得其内容,并通过:PreAuthenticatedAuthenticationToken包裹
			Authentication authentication = tokenExtractor.extract(request);
			
			// 2. 请求头里都没有内容
			if (authentication == null) {
				// 判断线程上下文中是否存在认证后的信息,如果存在,则需要清理线程上下文
				if (stateless && isAuthenticated()) {
					if (debug) {
						logger.debug("Clearing security context.");
					}
					SecurityContextHolder.clearContext();
				}
				if (debug) {
					logger.debug("No token in request, will continue chain.");
				}
			} else {
				
				request.setAttribute(OAuth2AuthenticationDetails.ACCESS_TOKEN_VALUE, authentication.getPrincipal());
				if (authentication instanceof AbstractAuthenticationToken) {
					AbstractAuthenticationToken needsDetails = (AbstractAuthenticationToken) authentication;
					// ********************************************************************************************
					// 通过:AuthenticationDetailsSource,构建认证后的详细信息(其实,可以在这一步,去加载用户所有的资源来着的)
					// ********************************************************************************************
					needsDetails.setDetails(authenticationDetailsSource.buildDetails(request));
				}
				
				// ********************************************************************************************
				// 调用了上面的:OAuth2AuthenticationManager,进行认证管理
				// ********************************************************************************************
				Authentication authResult = authenticationManager.authenticate(authentication);

				if (debug) {
					logger.debug("Authentication success: " + authResult);
				}
				
				eventPublisher.publishAuthenticationSuccess(authResult);
				// 把认证后的信息,重新设置到线程上下文里
				SecurityContextHolder.getContext().setAuthentication(authResult);

			}
		} catch (OAuth2Exception failed) {
			SecurityContextHolder.clearContext();
			if (debug) {
				logger.debug("Authentication request failed: " + failed);
			}
			
			eventPublisher.publishAuthenticationFailure(new BadCredentialsException(failed.getMessage(), failed),
					new PreAuthenticatedAuthenticationToken("access-token", "N/A"));
			authenticationEntryPoint.commence(request, response,
					new InsufficientAuthenticationException(failed.getMessage(), failed));
			return;
		}

		chain.doFilter(request, response);
	}

	private boolean isAuthenticated() {
		Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
		if (authentication == null || authentication instanceof AnonymousAuthenticationToken) {
			return false;
		}
		return true;
	}

	public void init(FilterConfig filterConfig) throws ServletException {
	}

	public void destroy() {
	}

	private static final class NullEventPublisher implements AuthenticationEventPublisher {
		public void publishAuthenticationFailure(AuthenticationException exception, Authentication authentication) {
		}

		public void publishAuthenticationSuccess(Authentication authentication) {
		}
	} // end NullEventPublisher
}
```
### (6). 总结
OAuth2AuthenticationProcessingFilter职责如下:  
+ 拦截HTTP请求,并获取:token信息.   
+ 委托给:AuthenticationDetailsSource丰富:Details信息.    
+ 委托给:OAuth2AuthenticationManager进行认证管理,鉴权部份依然是交给了,Spring Security(FilterSecurityInterceptor)进行处理.  