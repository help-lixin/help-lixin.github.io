---
layout: post
title: 'Spring Security与Oauth2整合源码之AuthorizationServerEndpointsConfiguration(十一)' 
date: 2021-10-11
author: 李新
tags:  SpringSecurity
---

### (1). 概述
在这一小篇,开始对Spring Security与Oauth2进行整合的源码进行剖析,需要找到Oauth2对Spring Security进行了哪些扩展.

### (2). 切入点(@EnableAuthorizationServer)
```
// Endpoint典型的请求入站消息配置
@Import({AuthorizationServerEndpointsConfiguration.class, AuthorizationServerSecurityConfiguration.class})
public @interface EnableAuthorizationServer {
}

```
### (3). AuthorizationServerEndpointsConfiguration
```
@Configuration
@Import(TokenKeyEndpointRegistrar.class)
public class AuthorizationServerEndpointsConfiguration {
	
	// ******************************************************************************************************
	// 1.  创建一个:AuthorizationServerEndpointsConfigurer
	// ******************************************************************************************************
	private AuthorizationServerEndpointsConfigurer endpoints = new AuthorizationServerEndpointsConfigurer();

	// ******************************************************************************************************
	// 2. 注入开发配置的:ClientDetailsService
	// ******************************************************************************************************
	@Autowired
	private ClientDetailsService clientDetailsService;

	
	// ******************************************************************************************************
	// 3. 注入开发配置的:AuthorizationServerConfigurer集合
	// ******************************************************************************************************
	@Autowired
	private List<AuthorizationServerConfigurer> configurers = Collections.emptyList();


	// ******************************************************************************************************
	// 4. 遍历开发配置的:AuthorizationServerConfigurer,设置到Global:AuthorizationServerEndpointsConfigurer
	// ******************************************************************************************************
	@PostConstruct
	public void init() {
		for (AuthorizationServerConfigurer configurer : configurers) {
			try {
				configurer.configure(endpoints);
			} catch (Exception e) {
				throw new IllegalStateException("Cannot configure enpdoints", e);
			}
		}
		
		// 设置:ClientDetailsService
		endpoints.setClientDetailsService(clientDetailsService);
	}

    // ******************************************************************************************************
	//    /oauth/confirm_access  请求处理
	// 5. 我一直以为Oauth2会像Spring Security一样,写一堆的Filter,结果:你会发现,超出你的想象.
	//    Oauth2压根就没有写Filter,而是直接利用注解(二次封装):@org.springframework.security.oauth2.provider.endpoint.FrameworkEndpoint,向SpringMVC中注册Mapping.
	// ******************************************************************************************************
	@Bean
	public AuthorizationEndpoint authorizationEndpoint() throws Exception {
		AuthorizationEndpoint authorizationEndpoint = new AuthorizationEndpoint();
		FrameworkEndpointHandlerMapping mapping = getEndpointsConfigurer().getFrameworkEndpointHandlerMapping();
		authorizationEndpoint.setUserApprovalPage(extractPath(mapping, "/oauth/confirm_access"));
		authorizationEndpoint.setProviderExceptionHandler(exceptionTranslator());
		authorizationEndpoint.setErrorPage(extractPath(mapping, "/oauth/error"));
		authorizationEndpoint.setTokenGranter(tokenGranter());
		authorizationEndpoint.setClientDetailsService(clientDetailsService);
		authorizationEndpoint.setAuthorizationCodeServices(authorizationCodeServices());
		authorizationEndpoint.setOAuth2RequestFactory(oauth2RequestFactory());
		authorizationEndpoint.setOAuth2RequestValidator(oauth2RequestValidator());
		authorizationEndpoint.setUserApprovalHandler(userApprovalHandler());
		return authorizationEndpoint;
	}


	// ******************************************************************************************************
	// 6. /oauth/token  请求处理(拿着code换access_token,以及拿着refresh_token换access_token)
	//    向SpringMVC中注册了一个Mapping
	// ******************************************************************************************************
	@Bean
	public TokenEndpoint tokenEndpoint() throws Exception {
		TokenEndpoint tokenEndpoint = new TokenEndpoint();
		tokenEndpoint.setClientDetailsService(clientDetailsService);
		tokenEndpoint.setProviderExceptionHandler(exceptionTranslator());
		tokenEndpoint.setTokenGranter(tokenGranter());
		tokenEndpoint.setOAuth2RequestFactory(oauth2RequestFactory());
		tokenEndpoint.setOAuth2RequestValidator(oauth2RequestValidator());
		tokenEndpoint.setAllowedRequestMethods(allowedTokenEndpointRequestMethods());
		return tokenEndpoint;
	}


	// ******************************************************************************************************
	// 7. /oauth/check_token 请求处理
	//    向SpringMVC中注册了一个Mapping
	// ******************************************************************************************************
	@Bean
	public CheckTokenEndpoint checkTokenEndpoint() {
		CheckTokenEndpoint endpoint = new CheckTokenEndpoint(getEndpointsConfigurer().getResourceServerTokenServices());
		endpoint.setAccessTokenConverter(getEndpointsConfigurer().getAccessTokenConverter());
		endpoint.setExceptionTranslator(exceptionTranslator());
		return endpoint;
	}


	// ******************************************************************************************************
	// 8. 认证成功后的URL处理:/oauth/confirm_access
	// ******************************************************************************************************
	@Bean
	public WhitelabelApprovalEndpoint whitelabelApprovalEndpoint() {
		return new WhitelabelApprovalEndpoint();
	}


	// ******************************************************************************************************
	// 9. 认证失败后的URL处理:/oauth/error
	// ******************************************************************************************************
	@Bean
	public WhitelabelErrorEndpoint whitelabelErrorEndpoint() {
		return new WhitelabelErrorEndpoint();
	}
	
} 	
```
### (4). 总结
在没有看源码之前,自己猜测是:Oauth2会自定义一堆的Filter,结果,超出你的预料,不过,这样有一个好处就是能扩展你的知识(毕竟:条条大路通罗马),有一点不解的是:Filter相比SpringMVC Mapping来说,Filter会更加的通用一些(比如:Struts与Oauth2结合怎么办).不知道Oauth2的作者是怎么想的? 