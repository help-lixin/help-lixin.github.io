---
layout: post
title: 'Spring Security源码之AccessDecisionVoter(十二)' 
date: 2021-10-11
author: 李新
tags:  SpringSecurity
---

### (1). 概述
在前面我们剖析AccessDecisionManager主要负责投票的决策管理,但是,内部实际又委托给了:AccessDecisionVoter进行投票

### (2). 看下AccessDecisionVoter接口能力
```
public interface AccessDecisionVoter<S> {
	// 允许访问
	int ACCESS_GRANTED = 1;
	// 弃权
	int ACCESS_ABSTAIN = 0;
	// 拒绝访问
	int ACCESS_DENIED = -1;
				
	// 判断该投票者是否能够对该安全对象配置的属性进行投票
	boolean supports(ConfigAttribute attribute);
	
    // 判断该投票者是否能够对该安全对象进行投票
	boolean supports(Class<?> clazz);
	
	// 投票
	int vote(Authentication authentication, S object, Collection<ConfigAttribute> attributes);
}
```
### (3). 查看AccessDecisionVoter接口实现
!["AccessDecisionVoter"](/assets/spring-security/imgs/spring-security-AccessDecisionVoter.png)

### (4). RoleVoter
```
public class RoleVoter implements AccessDecisionVoter<Object> {
	
	private String rolePrefix = "ROLE_";
		
	public void setRolePrefix(String rolePrefix) {
		this.rolePrefix = rolePrefix;
	}
	
	public String getRolePrefix() {
		return rolePrefix;
	}
	
	public boolean supports(ConfigAttribute attribute) {
		// 判断:attribute是否以指定的前缀开头
		if ((attribute.getAttribute() != null) && attribute.getAttribute().startsWith(getRolePrefix())) {
			return true;
		} else {
			return false;
		}
	} // end supports
		
	public boolean supports(Class<?> clazz) {
		return true;
	} // end supports
	
	// ********************************************************************************************************
	// 好像object没发挥什么作用哈
	// ********************************************************************************************************
	public int vote(Authentication authentication, Object object, Collection<ConfigAttribute> attributes) {
		// 1. 如果认证信息不存在,投票结果为:拒绝
		if(authentication == null) {
			return ACCESS_DENIED;  // 拒绝
		}
		
		// 2. 默认返回结果是:弃权
		int result = ACCESS_ABSTAIN;
		// 3. 获得认证后的:GrantedAuthority信息
		Collection<? extends GrantedAuthority> authorities = extractAuthorities(authentication);
		
		// 4. 遍历所有的:authorities
		for (ConfigAttribute attribute : attributes) {
			if (this.supports(attribute)) {
				result = ACCESS_DENIED;
				
				// Attempt to find a matching granted authority
				for (GrantedAuthority authority : authorities) {
					if (attribute.getAttribute().equals(authority.getAuthority())) {
						// 符合的情况下,则,进允许访问.
						return ACCESS_GRANTED;
					} // end if
				} // end for
			} // end if
		} // end for
		
		return result;
	}

	// 从authentication中获得:GrantedAuthority集合
	Collection<? extends GrantedAuthority> extractAuthorities(Authentication authentication) {
		return authentication.getAuthorities();
	} // end extractAuthorities
	
}
```
### (5). PreInvocationAuthorizationAdviceVoter
```
public class PreInvocationAuthorizationAdviceVoter implements
		AccessDecisionVoter<MethodInvocation> {
	protected final Log logger = LogFactory.getLog(getClass());

	private final PreInvocationAuthorizationAdvice preAdvice;

	public PreInvocationAuthorizationAdviceVoter(PreInvocationAuthorizationAdvice pre) {
		this.preAdvice = pre;
	}

	public boolean supports(ConfigAttribute attribute) {
		return attribute instanceof PreInvocationAttribute;
	}

	public boolean supports(Class<?> clazz) {
		return MethodInvocation.class.isAssignableFrom(clazz);
	}

	public int vote(Authentication authentication, MethodInvocation method,
			Collection<ConfigAttribute> attributes) {
		
		// 1. 遍历所有的:ConfigAttribute,看是不为:PreInvocationAttribute
		PreInvocationAttribute preAttr = findPreInvocationAttribute(attributes);

		if (preAttr == null) {
			// No expression based metadata, so abstain
			// 弃权
			return ACCESS_ABSTAIN;
		}

		// ******************************************************************************************
		// 2. 委托给:ExpressionBasedPreInvocationAdvice.before进行处理
		// ******************************************************************************************
		boolean allowed = preAdvice.before(authentication, method, preAttr);
		return allowed ? ACCESS_GRANTED : ACCESS_DENIED;
	}

	private PreInvocationAttribute findPreInvocationAttribute(
			Collection<ConfigAttribute> config) {
		for (ConfigAttribute attribute : config) {
			if (attribute instanceof PreInvocationAttribute) {
				return (PreInvocationAttribute) attribute;
			}
		}
		return null;
	}
}
```
### (6). 

### (7). 

### (8). 

### (9). 

### (10). 
 
### (11). 

