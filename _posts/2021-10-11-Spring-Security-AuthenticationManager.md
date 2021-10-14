---
layout: post
title: 'Spring Security源码之AuthenticationManager(九)' 
date: 2021-10-11
author: 李新
tags:  SpringSecurity
---

### (1). 概述
在前面,剖析了:AuthenticationProvider,那AuthenticationManager与AuthenticationProvider是什么关系呢?带着这个问题,我们来看:AuthenticationManager的源码.

### (2). 看下AuthenticationManager接口能力
```
package org.springframework.security.authentication;

import org.springframework.security.core.Authentication;
import org.springframework.security.core.AuthenticationException;


public interface AuthenticationManager {
	
	// 好简单,就一个认证方法
	Authentication authenticate(Authentication authentication) throws AuthenticationException;
	
}	
```
### (3). 看下AuthenticationManager的实现类
> 我们只关心:ProviderManager,其余的都是:Delegator对象.

!["AuthenticationManager"](/assets/spring-security/imgs/AuthenticationManager.jpg)

### (4). ProviderManager
```
public class ProviderManager implements AuthenticationManager, MessageSourceAware, InitializingBean {
	// *****************************************************************************
	// AuthenticationProvider集合
	// *****************************************************************************
   private List<AuthenticationProvider> providers = Collections.emptyList();
   
   // *****************************************************************************
   // 自己再包含着自己,这是很典型的组合模式.
   // *****************************************************************************
   private AuthenticationManager parent;
   
   
   public Authentication authenticate(Authentication authentication) throws AuthenticationException {
   		Class<? extends Authentication> toTest = authentication.getClass();
   		AuthenticationException lastException = null;
   		AuthenticationException parentException = null;
   		Authentication result = null;
   		Authentication parentResult = null;
   		boolean debug = logger.isDebugEnabled();
   
		// ******************************************************************************************
		// 遍历所有的:AuthenticationProvider
		// ******************************************************************************************
   		for (AuthenticationProvider provider : getProviders()) {
			
			// ******************************************************************************************
			// 1. 通过:AuthenticationProvider判断:Authentication类型是否支持,不支持的情况下,continue循环.
			// ******************************************************************************************
   			if (!provider.supports(toTest)) {
   				continue;
   			}
   
   			if (debug) {
   				logger.debug("Authentication attempt using " + provider.getClass().getName());
   			}
   
   			try {
				// ******************************************************************************************
				// 2. 调用:AuthenticationProvider.authenticate,验证用户名和密码
				// ******************************************************************************************
   				result = provider.authenticate(authentication);
				
				// 拷贝details到Authentication
   				if (result != null) {
   					copyDetails(authentication, result);
   					break;
   				}
   			} catch (AccountStatusException e) {
   				prepareException(e, authentication);
   				throw e;
   			} catch (InternalAuthenticationServiceException e) {
   				prepareException(e, authentication);
   				throw e;
   			} catch (AuthenticationException e) {
   				lastException = e;
   			}
   		} // end for
   
   		if (result == null && parent != null) {
   			// Allow the parent to try.
   			try {
				
				// 在这里用到了典型的组合模式来着的:parent虽然是ProviderManager,但是,是不同的实例对象来着的
				// parent == ProviderManager
   				result = parentResult = parent.authenticate(authentication);
   			} catch (ProviderNotFoundException e) {
   				
   			} catch (AuthenticationException e) {
   				lastException = parentException = e;
   			}
   		}
   
   		if (result != null) {
   			if (eraseCredentialsAfterAuthentication && (result instanceof CredentialsContainer)) {
   				// Authentication is complete. Remove credentials and other secret data
   				// from authentication
   				((CredentialsContainer) result).eraseCredentials();
   			}
   
   			// If the parent AuthenticationManager was attempted and successful than it will publish an AuthenticationSuccessEvent
   			// This check prevents a duplicate AuthenticationSuccessEvent if the parent AuthenticationManager already published it
   			if (parentResult == null) {
				// 发布事件
   				eventPublisher.publishAuthenticationSuccess(result);
   			}
			// 返回结果.
   			return result;
   		}
   
   		// Parent was null, or didn't authenticate (or throw an exception).
   
   		if (lastException == null) {
   			lastException = new ProviderNotFoundException(messages.getMessage("ProviderManager.providerNotFound",new Object[] { toTest.getName() }, "No AuthenticationProvider found for {0}"));
   		}
   
   		// If the parent AuthenticationManager was attempted and failed than it will publish an AbstractAuthenticationFailureEvent
   		// This check prevents a duplicate AbstractAuthenticationFailureEvent if the parent AuthenticationManager already published it
   		if (parentException == null) {
   			prepareException(lastException, authentication);
   		}
   
   		throw lastException;
   	} // end authenticate
}
```
### (5). AuthenticationManager职责
AuthenticationManager的职责如下:  
+ 遍历所有的:AuthenticationProvider
   - 通过AuthenticationProvider.supports方法,验证Authentication是否支持.   
   - 委派AuthenticationProvider.authenticate方法,校验用户名和密码是否正确.  

### (6). 总结
通过源码剖析,可以根据自己的情况来定制Spring Security的验证过程,定制开发步骤如下:  
+ 自定义一个Authentication(比如:TokenAuthentication)   
+ 自定义一个AuthenticationProvider,实现其方法,即可.      