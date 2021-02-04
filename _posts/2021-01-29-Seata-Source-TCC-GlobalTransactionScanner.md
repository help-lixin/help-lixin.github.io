---
layout: post
title: 'Seata  TCC全局事务之GlobalTransactionScanner(一)'
date: 2021-01-29
author: 李新
tags: Seata-TCC源码
---

### (1).  GlobalTransactionScanner
> GlobalTransactionScanner类在前面Seta-AT模式的源码时,有分析过了,在这里只关注有差别的地方.

### (2). GlobalTransactionScanner.wrapIfNecessary
```
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
	try {
		synchronized (PROXYED_SET) {
			if (PROXYED_SET.contains(beanName)) {
				return bean;
			}
			interceptor = null;
			
			// **********************************************************************
			// TCCBeanParserUtils.isTccAutoProxy的内容下一小节再去剖析.
			// **********************************************************************
			// 检查是否要代理只有满足以下三个条件: 
			//   sofa:reference(@Refrence) / dubbo:reference(@Refrence) /  @LocalTCC
			// 才会走拦截器
			if (TCCBeanParserUtils.isTccAutoProxy(bean, beanName, applicationContext)) {
				// *********************************************************************************
				// Seta注释还是挺全面的,这个interceptor(拦截器),只针对以下三种情况才有效:
				// sofa:reference(@Refrence)
				// dubbo:reference(@Refrence)
				// @LocalTCC
				// 意思是:生产者这一端,你要是打断点的话,压根不进这逻辑.
				// TCC interceptor, proxy bean of sofa:reference/dubbo:reference, and LocalTCC
				// *********************************************************************************
				interceptor = new TccActionInterceptor(TCCBeanParserUtils.getRemotingDesc(beanName));
				
				ConfigurationCache.addConfigListener(ConfigurationKeys.DISABLE_GLOBAL_TRANSACTION,
					(ConfigurationChangeListener)interceptor);
			} else {
				// 针对拥有:@GlobalTransactional和@GlobalTransactional的注解进行代码增强
				// ... ... 
			}

			LOGGER.info("Bean[{}] with name [{}] would use interceptor [{}]", bean.getClass().getName(), beanName, interceptor.getClass().getName());
			if (!AopUtils.isAopProxy(bean)) {
				bean = super.wrapIfNecessary(bean, beanName, cacheKey);
			} else {
				AdvisedSupport advised = SpringProxyUtils.getAdvisedSupport(bean);
				Advisor[] advisor = buildAdvisors(beanName, getAdvicesAndAdvisorsForBean(null, null, null));
				for (Advisor avr : advisor) {
					advised.addAdvisor(0, avr);
				}
			}
			PROXYED_SET.add(beanName);
			return bean;
		}
	} catch (Exception exx) {
		throw new RuntimeException(exx);
	}
}
```
### (3). 总结
> Spring在每次初始化一个Bean时,都会回调:GlobalTransactionScanner.wrapIfNecessary.  
> 1. TCC模式的拦截器是:TccActionInterceptor(但拦截器只在消费端生效@LocalTcc除外).        
> 2. 针对拥有:@GlobalTransactional和@GlobalTransactional的注解进行代码增强.   