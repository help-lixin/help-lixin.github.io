---
layout: post
title: 'Spring Security源码之AuthenticationProvider(八)' 
date: 2021-10-11
author: 李新
tags:  SpringSecurity
---

### (1). 概述
在聊AuthenticationManager之前,得先学习下AuthenticationProvider,因为:AuthenticationManager实际只是对:AuthenticationProvider的一种包裹来着的.

### (2). 看下AuthenticationProvider的类关系
```
org.springframework.security.authentication.AuthenticationProvider
	org.springframework.security.authentication.RememberMeAuthenticationProvider
	
	# 和数据库结合时,使用的:AuthenticationProvider
	org.springframework.security.authentication.dao.AbstractUserDetailsAuthenticationProvider
		org.springframework.security.authentication.dao.DaoAuthenticationProvider
	
	org.springframework.security.authentication.rcp.RemoteAuthenticationProvider
	org.springframework.security.web.authentication.preauth.PreAuthenticatedAuthenticationProvider
	org.springframework.security.access.intercept.RunAsImplAuthenticationProvider
	org.springframework.security.authentication.AnonymousAuthenticationProvider
	org.springframework.security.config.annotation.web.configurers.oauth2.client.OidcAuthenticationRequestChecker
	org.springframework.security.authentication.jaas.AbstractJaasAuthenticationProvider
		org.springframework.security.authentication.jaas.JaasAuthenticationProvider
		org.springframework.security.authentication.jaas.DefaultJaasAuthenticationProvider
```
### (3). 看下AuthenticationProvider有哪些接口能力
> 从接口的能力上,我们能看出来:AuthenticationProvider是负责认证.

```
package org.springframework.security.authentication;

import org.springframework.security.core.Authentication;
import org.springframework.security.core.AuthenticationException;

public interface AuthenticationProvider {
	
	// 1. 判断当前的AuthenticationProvider,是否支持:Authentication
	boolean supports(Class<?> authentication);
	
	// 2. 进行认证管理
	Authentication authenticate(Authentication authentication) throws AuthenticationException;
}
```
### (4). AbstractUserDetailsAuthenticationProvider
> 我们重点研究:DaoAuthenticationProvider,而,AbstractUserDetailsAuthenticationProvider是它的父类,所以,先看下父类的调用链

```
public abstract class AbstractUserDetailsAuthenticationProvider 
                implements AuthenticationProvider, InitializingBean, MessageSourceAware {
	protected MessageSourceAccessor messages = SpringSecurityMessageSource.getAccessor();
	private UserCache userCache = new NullUserCache();
	private boolean forcePrincipalAsString = false;
	protected boolean hideUserNotFoundExceptions = true;
	private UserDetailsChecker preAuthenticationChecks = new DefaultPreAuthenticationChecks();
	private UserDetailsChecker postAuthenticationChecks = new DefaultPostAuthenticationChecks();
	private GrantedAuthoritiesMapper authoritiesMapper = new NullAuthoritiesMapper();
	
	
	public Authentication authenticate(Authentication authentication) throws AuthenticationException {
		// 验证下是否为:UsernamePasswordAuthenticationToken
		Assert.isInstanceOf(UsernamePasswordAuthenticationToken.class, authentication, messages.getMessage("AbstractUserDetailsAuthenticationProvider.onlySupports" , "Only UsernamePasswordAuthenticationToken is supported"));
		
		// 从Authentication中拿出:用户名
		// Determine username
		String username = (authentication.getPrincipal() == null) ? "NONE_PROVIDED" : authentication.getName();
		
		// **********************************************************************************
		// 1. 根据:用户名称(username),先从缓存中,获得:UserDetails(认证成功后的用户载体数据)
		// **********************************************************************************
		boolean cacheWasUsed = true;
		UserDetails user = this.userCache.getUserFromCache(username);

		// **********************************************************************************
		// 2. 缓存中不存在的情况下
		// **********************************************************************************
		if (user == null) {
			cacheWasUsed = false;

			try {
				// **********************************************************************************
				// 3. 委托给子类(DaoAuthenticationProvider),检索用户信息
				// **********************************************************************************
				user = retrieveUser(username, (UsernamePasswordAuthenticationToken) authentication);
			} catch (UsernameNotFoundException notFound) {
				logger.debug("User '" + username + "' not found");
				
				if (hideUserNotFoundExceptions) {
					throw new BadCredentialsException(messages.getMessage("AbstractUserDetailsAuthenticationProvider.badCredentials","Bad credentials"));
				} else {
					throw notFound;
				}
			}
			Assert.notNull(user, "retrieveUser returned null - a violation of the interface contract");
		}

		try {
			// 认证之前检查UserDetails对象
			preAuthenticationChecks.check(user);
			
			// **********************************************************************************
			// 4. 委派给子类(DaoAuthenticationProvider),拿着表单中的密码和DAO检索出来的密码进行比较,判断是否相等
			// **********************************************************************************
			additionalAuthenticationChecks(user, (UsernamePasswordAuthenticationToken) authentication);
		} catch (AuthenticationException exception) {
			if (cacheWasUsed) {  // 如果有开启了缓存,检查不通过的情况下,再次重试进行用户检索
				cacheWasUsed = false;
				user = retrieveUser(username , (UsernamePasswordAuthenticationToken) authentication);
				preAuthenticationChecks.check(user);
				additionalAuthenticationChecks(user , (UsernamePasswordAuthenticationToken) authentication);
			} else {
				throw exception;
			}
		}

        // 认证之前检查UserDetails对象
		postAuthenticationChecks.check(user);
		
		// 设置到缓存里
		if (!cacheWasUsed) {
			this.userCache.putUserInCache(user);
		}

		Object principalToReturn = user;

		// 是否强制,把username转换到:principalToReturn
		if (forcePrincipalAsString) {
			principalToReturn = user.getUsername();
		}

		// **********************************************************************************
		// 5. 创建:Authentication(UsernamePasswordAuthenticationToken)
		// **********************************************************************************
		return createSuccessAuthentication(principalToReturn, authentication, user);
	} // end authenticate
	
	
	protected Authentication createSuccessAuthentication(Object principal,
				Authentication authentication, UserDetails user) {
		UsernamePasswordAuthenticationToken result = new UsernamePasswordAuthenticationToken(principal, authentication.getCredentials(),authoritiesMapper.mapAuthorities(user.getAuthorities()));
		result.setDetails(authentication.getDetails());
		return result;
	} // end createSuccessAuthentication
}
```
### (5). DaoAuthenticationProvider
```
public class DaoAuthenticationProvider extends AbstractUserDetailsAuthenticationProvider {
	
	// 对密码的编码和解码处理.
	// DelegatingPasswordEncoder
	private PasswordEncoder passwordEncoder;
	
	// **********************************************************************
	// UserDetailsService主要负责,检索用户信息
	// **********************************************************************
	private UserDetailsService userDetailsService;
	
	
	// 检索用户信息
	protected final UserDetails retrieveUser(String username , UsernamePasswordAuthenticationToken authentication) throws AuthenticationException {
		prepareTimingAttackProtection();
		try {
			// ************************************************************************************************
			// 委托给了:UserDetailsService,根据用户名去加载用户详细信息出来.
			// ************************************************************************************************
			UserDetails loadedUser = this.getUserDetailsService().loadUserByUsername(username);
			
			if (loadedUser == null) { // 用户不存在,则抛出异常
				throw new InternalAuthenticationServiceException("UserDetailsService returned null, which is an interface contract violation");
			}
			return loadedUser;
		} catch (UsernameNotFoundException ex) {
			mitigateAgainstTimingAttack(authentication);
			throw ex;
		} catch (InternalAuthenticationServiceException ex) {
			throw ex;
		} catch (Exception ex) {
			throw new InternalAuthenticationServiceException(ex.getMessage(), ex);
		}
	} // end retrieveUser
	
	
	// 认证检查
	protected void additionalAuthenticationChecks(UserDetails userDetails, UsernamePasswordAuthenticationToken authentication) throws AuthenticationException {
		if (authentication.getCredentials() == null) {
			logger.debug("Authentication failed: no credentials provided");

			throw new BadCredentialsException(messages.getMessage("AbstractUserDetailsAuthenticationProvider.badCredentials","Bad credentials"));
		}

		String presentedPassword = authentication.getCredentials().toString();
		
		// **************************************************************************************************
		// 验证密码是否相同,如果,不同的情况下,抛出异常:BadCredentialsException
		// **************************************************************************************************
		if (!passwordEncoder.matches(presentedPassword, userDetails.getPassword())) {
			logger.debug("Authentication failed: password does not match stored value");

			throw new BadCredentialsException(messages.getMessage("AbstractUserDetailsAuthenticationProvider.badCredentials","Bad credentials"));
		}
	}
	
}
```
### (6). 总结
AuthenticationManager与AuthenticationProvider有什么样的关联呢?下一小节,会详细进行剖析,在这里先聊下AuthenticationProvider的职责:  
+ 根据用户名(username),从DB中检索出:UserDetails.  
+ 通过PasswordEncoder对UserDetails中的密码与表单提交过来的密码进行验证.   
+ 认证通过后,创建:Authentication对象(包裹着用户名/密码/详细信息),并返回.   