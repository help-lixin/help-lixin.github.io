---
layout: post
title: 'Seata GlobalTransactionScanner(一)'
date: 2021-01-28
author: 李新
tags: Seata源码
---
### (1). spring-cloud-alibaba-seata-2.1.0.RELEASE(spring.factories)
> 查看seata是如何与Spring Boot结合的(找到入口文件):    
> spring-cloud-alibaba-seata-2.1.0.RELEASE.jar/META-INF/spring.factories      

```
# SeataRestTemplateAutoConfiguration
# SeataHandlerInterceptorConfiguration 
# SeataFeignClientAutoConfiguration
# SeataHystrixAutoConfiguration
# 以上四种,都是解决在不同环境(RestTemplate/MVC/Feign/Hystrix)情况下,把ThreadLocal中的TX_XID,传递给下一链路.

# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.alibaba.cloud.seata.rest.SeataRestTemplateAutoConfiguration,\
com.alibaba.cloud.seata.web.SeataHandlerInterceptorConfiguration,\
com.alibaba.cloud.seata.GlobalTransactionAutoConfiguration,\
com.alibaba.cloud.seata.feign.SeataFeignClientAutoConfiguration,\
com.alibaba.cloud.seata.feign.hystrix.SeataHystrixAutoConfiguration
```

### (2). SeataProperties
> SeataProperties是Seata的配置信息载体.

```
@ConfigurationProperties("spring.cloud.alibaba.seata")
public class SeataProperties {
	private String txServiceGroup;
	
	public String getTxServiceGroup() {
		return txServiceGroup;
	}
	
	public void setTxServiceGroup(String txServiceGroup) {
		this.txServiceGroup = txServiceGroup;
	}
}
```
### (3). GlobalTransactionAutoConfiguration
```
package com.alibaba.cloud.seata;

import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.util.StringUtils;

import io.seata.spring.annotation.GlobalTransactionScanner;

@Configuration
@EnableConfigurationProperties(SeataProperties.class)
public class GlobalTransactionAutoConfiguration {

	private final ApplicationContext applicationContext;

	private final SeataProperties seataProperties;

	public GlobalTransactionAutoConfiguration(ApplicationContext applicationContext,
			SeataProperties seataProperties) {
		this.applicationContext = applicationContext;
		this.seataProperties = seataProperties;
	}

	@Bean
	public GlobalTransactionScanner globalTransactionScanner() {
		// 获得应用程序的名称
		String applicationName = applicationContext.getEnvironment()
				.getProperty("spring.application.name");
        // 获得配置信息:spring.cloud.alibaba.seata.tx-service-group
		String txServiceGroup = seataProperties.getTxServiceGroup();
		
		// 如果没有配置该属性,则txServiceGroup=微服服务名称+-fescar-service-group
		if (StringUtils.isEmpty(txServiceGroup)) {
			// *************************************************************
			// 微服务名称(account-service) + -fescar-service-group
			// 例如:account-service-fescar-service-group
			// 建议不要配置:spring.cloud.alibaba.seata.tx-service-group
			// *************************************************************
			txServiceGroup = applicationName + "-fescar-service-group";
			seataProperties.setTxServiceGroup(txServiceGroup);
		}
		
		// ******************************************************************
		// Seata的入口类
		// ******************************************************************
		return new GlobalTransactionScanner(applicationName, txServiceGroup);
	}
}
```

### (5). GlobalTransactionScanner继承结构图

!["GlobalTransactionScanner继承结构图"](/assets/seata/imgs/seata-main-GlobalTransactionScanner.jpg)

### (6). GlobalTransactionScanner.afterPropertiesSet
> 由于GlobalTransactionScanner实现了:InitializingBean,
> 所以,在Spring实例化对象之后,会回调:afterPropertiesSet方法.

```
// domain
private final AtomicBoolean initialized = new AtomicBoolean(false);

// methods 
public void afterPropertiesSet() {
	
	ConfigurationCache.addConfigListener(ConfigurationKeys.DISABLE_GLOBAL_TRANSACTION,
		(ConfigurationChangeListener)this);
	if (disableGlobalTransaction) {
		if (LOGGER.isInfoEnabled()) {
			LOGGER.info("Global transaction is disabled.");
		}
		return;
	}
	// ****************************************************
	// 确保:afterPropertiesSet方法只运行一次
	// ****************************************************
	if (initialized.compareAndSet(false, true)) {
		initClient();
	}
}
```
### (7). GlobalTransactionScanner.initClient
```
private void initClient() {
	// ... ...
	
	// **************************************************************
	// 初始化:TM(事务管理器)
	// **************************************************************
	//init TM
	TMClient.init(applicationId, txServiceGroup, accessKey, secretKey);
	
	
	// **************************************************************
	//init RM
	// 初始化:RM(分支事务资源管理器)
	// **************************************************************
	RMClient.init(applicationId, txServiceGroup);
	// ... ...
	
	// 注册Spring关闭时的回调函数(钩子函数)
	registerSpringShutdownHook();

}
```
### (8). GlobalTransactionScanner.wrapIfNecessary
> AbstractAutoProxyCreator.postProcessAfterInitialization会回调:GlobalTransactionScanner.wrapIfNecessary方法.  

```
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
	try {
		synchronized (PROXYED_SET) {
			if (PROXYED_SET.contains(beanName)) {
				return bean;
			}
			interceptor = null;
			
			
			// TCC代理处理
			//check TCC proxy
			if (TCCBeanParserUtils.isTccAutoProxy(bean, beanName, applicationContext)) {
				//TCC interceptor, proxy bean of sofa:reference/dubbo:reference, and LocalTCC
				interceptor = new TccActionInterceptor(TCCBeanParserUtils.getRemotingDesc(beanName));
				ConfigurationCache.addConfigListener(ConfigurationKeys.DISABLE_GLOBAL_TRANSACTION,
					(ConfigurationChangeListener)interceptor);
			} else {
				
				Class<?> serviceInterface = SpringProxyUtils.findTargetClass(bean);
				Class<?>[] interfacesIfJdk = SpringProxyUtils.findInterfaces(bean);

				// 如果类上或者接口上不包含:@GlobalTransactional或@GlobalLock
				// 则直接返回到父类:AbstractAutoProxyCreator.postProcessAfterInitialization方法里.
				if (!existsAnnotation(new Class[]{serviceInterface})
					&& !existsAnnotation(interfacesIfJdk)) {
					return bean;
				}

				// 代码能走到这里:代表类或者接口上有注解:@GlobalTransactional或@GlobalLock
				if (interceptor == null) {
					if (globalTransactionalInterceptor == null) {
						// *********************************************************
						// 创建:GlobalTransactionalInterceptor
						// 后面留一节,专门讲解这里的逻辑
						// *********************************************************
						globalTransactionalInterceptor = new GlobalTransactionalInterceptor(failureHandlerHook);
						
						ConfigurationCache.addConfigListener(
							ConfigurationKeys.DISABLE_GLOBAL_TRANSACTION,
							(ConfigurationChangeListener)globalTransactionalInterceptor);
					}
					interceptor = globalTransactionalInterceptor;
				}
			}

			LOGGER.info("Bean[{}] with name [{}] would use interceptor [{}]", bean.getClass().getName(), beanName, interceptor.getClass().getName());
			// 如果这个对象不是一个代理对象.则通过:wrapIfNecessary方法进行AOP的代理.
			if (!AopUtils.isAopProxy(bean)) {
				bean = super.wrapIfNecessary(bean, beanName, cacheKey);
			} else {
				AdvisedSupport advised = SpringProxyUtils.getAdvisedSupport(bean);
				Advisor[] advisor = buildAdvisors(beanName, getAdvicesAndAdvisorsForBean(null, null, null));
				for (Advisor avr : advisor) {
					advised.addAdvisor(0, avr);
				}
			}
			// 添加到临时集合中.
			PROXYED_SET.add(beanName);
			// 返回的是一个经过代理之后的bean
			return bean;
		}
	} catch (Exception exx) {
		throw new RuntimeException(exx);
	}
} // end wrapIfNecessary



// AbstractAutoProxyCreator.wrapIfNecessary方法会回调该方法,获得要织入的代码逻辑(MethodInterceptor)
// AT模式下是:GlobalTransactionalInterceptor
// TCC模式下是:TccActionInterceptor
protected Object[] getAdvicesAndAdvisorsForBean(Class beanClass, String beanName, TargetSource customTargetSource)
            throws BeansException {
	return new Object[]{interceptor};
} //end getAdvicesAndAdvisorsForBean
```
### (9). 总结
> GlobalTransactionScanner被Spring容器托管后,会做如下几件事:
> 1. 初始化TM(事物管理器). 
> 2. 初始化RM(资源管理器). 
> 3. 扫描Spring容器里所有的Bean,如果Bean上有指定的注解(@GlobalTransactional/@GlobalLock),则,对这个Bean进行AOP代理.   
> 4. TCC模式下代理的逻辑代码是:TccActionInterceptor.   
> 5. AT模式下代理的逻辑工码是:GlobalTransactionalInterceptor.