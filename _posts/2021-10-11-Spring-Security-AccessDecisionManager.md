---
layout: post
title: 'Spring Security源码之AccessDecisionManager(十一)' 
date: 2021-10-11
author: 李新
tags:  SpringSecurity
---

### (1). 概述
在前面我们剖析FilterSecurityInterceptor主要负责鉴权功能,但是,在内部,实则又委托给了:AccessDecisionManager进行,所以,这一篇主要是对:AccessDecisionManager源码进行剖析.   

### (2). 看下AccessDecisionManager接口能力
```
package org.springframework.security.access;

import java.util.Collection;

import org.springframework.security.authentication.InsufficientAuthenticationException;
import org.springframework.security.core.Authentication;

/**
 * 鉴权决策(Decision)管理
 * 
 */
public interface AccessDecisionManager {
	
	// 决定(验证)
	void decide(Authentication authentication, Object object, Collection<ConfigAttribute> configAttributes) throws AccessDeniedException, InsufficientAuthenticationException;
	
	// 判断该投票者是否能够对该安全对象配置的属性进行投票
	boolean supports(ConfigAttribute attribute);
	
	// 断该投票者是否能够对该安全对象进行投票
	boolean supports(Class<?> clazz);
}
```
### (3). 看下AccessDecisionManager的实现类
!["AccessDecisionManager的实现类"](/assets/spring-security/imgs/spring-security-AccessDecisionManager.png)
### (4). AbstractAccessDecisionManager
```
public abstract class AbstractAccessDecisionManager implements AccessDecisionManager, InitializingBean, MessageSourceAware { 
	// ***********************************************************************************************
	// 1. 访问决策投票者,实际上是由这些投票者进行投票,然后委托给了不同的AccessDecisionManager根据投票者返回结果
	// ***********************************************************************************************
	private List<AccessDecisionVoter<? extends Object>> decisionVoters;
	
	
	protected MessageSourceAccessor messages = SpringSecurityMessageSource.getAccessor();

	// 感觉这玩意儿是代表:是否可以一票否决权哈
	private boolean allowIfAllAbstainDecisions = false;

	// ***********************************************************************************************
	// 2. 参与决策投票,是必须在构造时注入的
	// ***********************************************************************************************
	protected AbstractAccessDecisionManager(List<AccessDecisionVoter<? extends Object>> decisionVoters) {
		Assert.notEmpty(decisionVoters, "A list of AccessDecisionVoters is required");
		this.decisionVoters = decisionVoters;
	}

	public void afterPropertiesSet() throws Exception {
		Assert.notEmpty(this.decisionVoters, "A list of AccessDecisionVoters is required");
		Assert.notNull(this.messages, "A message source must be set");
	}

	protected final void checkAllowIfAllAbstainDecisions() {
		if (!this.isAllowIfAllAbstainDecisions()) {
			throw new AccessDeniedException(messages.getMessage("AbstractAccessDecisionManager.accessDenied", "Access is denied"));
		}
	}

	
	// 3. 验证AccessDecisionVoter是否支持:ConfigAttribute
	public boolean supports(ConfigAttribute attribute) {
		for (AccessDecisionVoter voter : this.decisionVoters) {
			if (voter.supports(attribute)) {
				return true;
			}
		}
		return false;
	}

	
	// 4. 验证AccessDecisionVoter是否支持:Class<?>
	public boolean supports(Class<?> clazz) {
		for (AccessDecisionVoter voter : this.decisionVoters) {
			if (!voter.supports(clazz)) {
				return false;
			}
		}
		return true;
	}
}
```
### (5). AffirmativeBased
> AffirmativeBased的职责:  
+ 只要有一票是:ACCESS_GRANTED,则授矛访问权限
+ 只要有一票是:ACCESS_DENIED,则拒绝访问.  

```
public class AffirmativeBased extends AbstractAccessDecisionManager {
   
	public void decide(Authentication authentication, Object object,
				Collection<ConfigAttribute> configAttributes) throws AccessDeniedException {
		
		// 拒绝访问的统计数量
		int deny = 0;
		
		// 1. 遍历投票者:AccessDecisionVoter
		for (AccessDecisionVoter voter : getDecisionVoters()) {
			// 2.  投票
			int result = voter.vote(authentication, object, configAttributes);
			if (logger.isDebugEnabled()) {
				logger.debug("Voter: " + voter + ", returned: " + result);
			}
			
			// AccessDecisionVoter.ACCESS_ABSTAIN   --> 弃权
			// AccessDecisionVoter.ACCESS_GRANTED   --> 允许
			// AccessDecisionVoter.ACCESS_DENIED    --> 拒绝
			
			// 3. 只要有一个投票者投了授权访问则停止遍历,认为允许访问
			switch (result) {
				case AccessDecisionVoter.ACCESS_GRANTED:  // 允许
					return;
				case AccessDecisionVoter.ACCESS_DENIED:   // 拒绝
					deny++;
					break;
				default:
					break;
			}
		} // end for

		//	只要有一个是拒绝的,则抛出异常(AccessDeniedException)
		if (deny > 0) {
			throw new AccessDeniedException(messages.getMessage("AbstractAccessDecisionManager.accessDenied", "Access is denied"));
		}
		
		// 最终还有一个一票否决权,不太理解
		// To get this far, every AccessDecisionVoter abstained
		checkAllowIfAllAbstainDecisions();
	} // end decide
}	
```
### (6). ConsensusBased
> ConsensusBased的职责:  
+ 允许访问的统计数量 > 拒绝访问的统计数量,则授矛访问权限.   
+ 拒绝访问的统计数量 > 允许访问的统计数量,则拒绝访问.  
+ 允许访问的统计数量 和 拒绝访问的统计数量,持平的情况下.
  - 配置了,否允许投授权访问和投拒绝访问的票数一致,则授矛访问权限.   
  - 配置了,不允许投授权访问和投拒绝访问的票数一致,则拒绝访问.

```
public class ConsensusBased extends AbstractAccessDecisionManager {
	
	// 是否允许投授权访问和投拒绝访问的票数一致
	private boolean allowIfEqualGrantedDeniedDecisions = true;
	
	public void decide(Authentication authentication, Object object, Collection<ConfigAttribute> configAttributes) throws AccessDeniedException {
		// AccessDecisionVoter.ACCESS_ABSTAIN   --> 弃权
		// AccessDecisionVoter.ACCESS_GRANTED   --> 允许
		// AccessDecisionVoter.ACCESS_DENIED    --> 拒绝
		
		// 允放访问的统计数量
		int grant = 0;
		// 拒绝访问的统计数量
		int deny = 0;
		// 弃权访问的统计数量
		int abstain = 0;
		
		// 1. 遍历投票者:AccessDecisionVoter
		for (AccessDecisionVoter voter : getDecisionVoters()) {
			
			// 2. 投票
			int result = voter.vote(authentication, object, configAttributes);

			if (logger.isDebugEnabled()) {
				logger.debug("Voter: " + voter + ", returned: " + result);
			}

			// 3. 对所有的投票结果进行累加
			switch (result) {
				case AccessDecisionVoter.ACCESS_GRANTED: 
					grant++;

					break;

				case AccessDecisionVoter.ACCESS_DENIED:
					deny++;

					break;

				default:
					abstain++;

					break;
			}
		} // end for

		// 允许访问的统计数量 > 拒绝访问的统计数量
		if (grant > deny) {
			return;
		}
		
		// 拒绝访问的统计数量 > 允许访问的统计数量 ,抛出异常:AccessDeniedException
		if (deny > grant) {
			throw new AccessDeniedException(messages.getMessage("AbstractAccessDecisionManager.accessDenied", "Access is denied"));
		}
		
		// 允许访问的统计数量 和 拒绝访问的统计数量 持平的情况下
		if ((grant == deny) && (grant != 0)) {
			// 当配置了,否允许投授权访问和投拒绝访问的票数一致,则授权允许访问,否则,抛出异常(AccessDeniedException).
			if (this.allowIfEqualGrantedDeniedDecisions) {
				return;
			}
			else {
				throw new AccessDeniedException(messages.getMessage("AbstractAccessDecisionManager.accessDenied", "Access is denied"));
			}
		} // end if
		
		
		// To get this far, every AccessDecisionVoter abstained
		checkAllowIfAllAbstainDecisions();
	} // end decide
	
}	
```
### (7). UnanimousBased
> UnanimousBased的职责如下:  
+ 只要有一个投票者给其中配置的属性投了拒绝访问则认为不允许访问.   

```
public class UnanimousBased extends AbstractAccessDecisionManager {
	
	public void decide(Authentication authentication, Object object, Collection<ConfigAttribute> attributes) throws AccessDeniedException {
        // AccessDecisionVoter.ACCESS_ABSTAIN   --> 弃权
        // AccessDecisionVoter.ACCESS_GRANTED   --> 允许
        // AccessDecisionVoter.ACCESS_DENIED    --> 拒绝
		
		// 允许访问的统计数量
		int grant = 0;
		// 弃权统计数量
		int abstain = 0;

		List<ConfigAttribute> singleAttributeList = new ArrayList<>(1);
		singleAttributeList.add(null);

		// 
		for (ConfigAttribute attribute : attributes) {
			singleAttributeList.set(0, attribute);

			// 
			for (AccessDecisionVoter voter : getDecisionVoters()) {
				int result = voter.vote(authentication, object, singleAttributeList);

				if (logger.isDebugEnabled()) {
					logger.debug("Voter: " + voter + ", returned: " + result);
				}

				switch (result) {
					case AccessDecisionVoter.ACCESS_GRANTED:  // 允许
						grant++;                              // 累加
						break;
					case AccessDecisionVoter.ACCESS_DENIED:   // 拒绝
						// 只要遇到拒绝的访问,直接抛出异常.
						throw new AccessDeniedException(messages.getMessage("AbstractAccessDecisionManager.accessDenied", "Access is denied"));

					default:  // 弃权
						abstain++;                            // 累加
						break;
				} // end switch
				
			} // end for
		} // end for
		
		
		// To get this far, there were no deny votes
		if (grant > 0) {
			return;
		}

		// To get this far, every AccessDecisionVoter abstained
		checkAllowIfAllAbstainDecisions();
	} 
	
}	
```
### (8). 总结
AccessDecisionManager的内部,实际会委托给一批AccessDecisionVoter进行投票管理.   