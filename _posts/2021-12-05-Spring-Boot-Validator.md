---
layout: post
title: 'Spring Boot与Validator整合以及源码剖析' 
date: 2021-12-05
author: 李新
tags:  SpringBoot 
---

### (1). 概述
以前都是用Spring(XML)去配置Validator(fluent-validator),没有太细节的看这部份的源码,最主要的原因是:在我脑里无法不过就是AOP拦截(生成动态代理)而已.    
但是,今天在与Spring Boot整合时,被坑了,所以,特意记录下来.   

### (2). 业务代码配置@Validated注解
```
@RequestMapping("/platformInfo")
// ***********************************************************************
// @Validated一定要放在类级别,不支持在方法级别的AOP处理.
// 在我记忆中:fluent-validator是支持方法级别的,所以,坑惨了.
// ***********************************************************************
@Validated
public class PlatformInfoServiceController {
	
	@GetMapping("/validator")
	// ***********************************************************************
	// 注意:刚开始@Validated我是放在方法上的.
	// ***********************************************************************
	// @Validated
	public String validator(@NotBlank @RequestParam(value = "message", required = false) String message) {
		return "SUCCESS";
	}
}
```
### (3). 配置Validator提供者
```
@Bean
public Validator validator() {
	return Validation
			.byProvider(HibernateValidator.class)
			.configure()
			//快速返回模式，有一个验证失败立即返回错误信息
			.failFast(true)
			.buildValidatorFactory()
			.getValidator();
}
```
### (4). 测试结果
```
# 不论怎么测试,发现都不会走验证,请求,都是直接进入业务代码里.  
lixin-macbook:~ lixin$ curl http://localhost:8080/platformInfo/validator
```
### (5). 看源码:ValidationAutoConfiguration
```
package org.springframework.boot.autoconfigure.validation;

import javax.validation.Validator;
import javax.validation.executable.ExecutableValidator;

import org.springframework.beans.factory.config.BeanDefinition;
import org.springframework.boot.autoconfigure.EnableAutoConfiguration;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnResource;
import org.springframework.boot.validation.MessageInterpolatorFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Import;
import org.springframework.context.annotation.Lazy;
import org.springframework.context.annotation.Role;
import org.springframework.core.env.Environment;
import org.springframework.validation.beanvalidation.LocalValidatorFactoryBean;
import org.springframework.validation.beanvalidation.MethodValidationPostProcessor;

@Configuration
@ConditionalOnClass(ExecutableValidator.class)
@ConditionalOnResource(resources = "classpath:META-INF/services/javax.validation.spi.ValidationProvider")
@Import(PrimaryDefaultValidatorPostProcessor.class)
public class ValidationAutoConfiguration {

    // ******************************************************************
	// 2. 如果应用中有自定义:Validator,则当前这个配置是失效的.
	// ******************************************************************
	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	@ConditionalOnMissingBean(Validator.class)
	public static LocalValidatorFactoryBean defaultValidator() {
		LocalValidatorFactoryBean factoryBean = new LocalValidatorFactoryBean();
		MessageInterpolatorFactory interpolatorFactory = new MessageInterpolatorFactory();
		factoryBean.setMessageInterpolator(interpolatorFactory.getObject());
		return factoryBean;
	}

	// ******************************************************************
	// 1. 从这个名字上就能看出来:MethodValidationPostProcessor
	//    这应该是个:BeanPostProcessor
	// ******************************************************************
	@Bean
	@ConditionalOnMissingBean
	public static MethodValidationPostProcessor methodValidationPostProcessor(
			Environment environment, @Lazy Validator validator) {
		MethodValidationPostProcessor processor = new MethodValidationPostProcessor();
		boolean proxyTargetClass = environment
				.getProperty("spring.aop.proxy-target-class", Boolean.class, true);
		processor.setProxyTargetClass(proxyTargetClass);
		processor.setValidator(validator);
		return processor;
	}

}
```
### (6). 看源码:MethodValidationPostProcessor
```
public class MethodValidationPostProcessor extends AbstractBeanFactoryAwareAdvisingPostProcessor
		implements InitializingBean {
	
	private Class<? extends Annotation> validatedAnnotationType = Validated.class;

	@Nullable
	private Validator validator;
	
	public void afterPropertiesSet() {
		// 1. 定义切面,要针对哪个注解,进行拦截.
		Pointcut pointcut = new AnnotationMatchingPointcut(this.validatedAnnotationType, true);
		// 2. 定义切入点
		this.advisor = new DefaultPointcutAdvisor(pointcut, createMethodValidationAdvice(this.validator));
	} // end afterPropertiesSet
	
	protected Advice createMethodValidationAdvice(@Nullable Validator validator) {
		// *****************************************************
		// MethodValidationInterceptor是AOP的逻辑代码
		// *****************************************************
		return (validator != null ? new MethodValidationInterceptor(validator) : new MethodValidationInterceptor());
	} // end createMethodValidationAdvice
}	
```
### (7). 看源码:MethodValidationInterceptor
```
public class MethodValidationInterceptor implements MethodInterceptor {
	public Object invoke(MethodInvocation invocation) throws Throwable {
		
		// 验证方法上是否有注解@Validated,没有的情况下,直接跳过了.
		// Avoid Validator invocation on FactoryBean.getObjectType/isSingleton
		if (isFactoryBeanMetadataMethod(invocation.getMethod())) {
			return invocation.proceed();
		}
		
		// group定义
		Class<?>[] groups = determineValidationGroups(invocation);

		// Standard Bean Validation 1.1 API
		ExecutableValidator execVal = this.validator.forExecutables();
		Method methodToValidate = invocation.getMethod();
		Set<ConstraintViolation<Object>> result;

		try {
			// 验证参数
			result = execVal.validateParameters(invocation.getThis(), methodToValidate, invocation.getArguments(), groups);
		} catch (IllegalArgumentException ex) {
			methodToValidate = BridgeMethodResolver.findBridgedMethod(ClassUtils.getMostSpecificMethod(invocation.getMethod(), invocation.getThis().getClass()));
			result = execVal.validateParameters(invocation.getThis(), methodToValidate, invocation.getArguments(), groups);
		}
		
		// ****************************************************************************
		// 验证结果集里有内容的情况下,代表验证失败,会抛出异常(ConstraintViolationException)
		// ****************************************************************************
		if (!result.isEmpty()) {
			throw new ConstraintViolationException(result);
		}
		
		
		Object returnValue = invocation.proceed();

		result = execVal.validateReturnValue(invocation.getThis(), methodToValidate, returnValue, groups);
		if (!result.isEmpty()) {
			throw new ConstraintViolationException(result);
		}

		return returnValue;
	} // end invoke
}	
```
### (8). 总结
@Validated的注解定义还是挺坑的,支持(ElementType.TYPE, ElementType.METHOD, ElementType.PARAMETER),但是,却只能放在类级别上,否则,AOP不会进行拦截.    
其实,我在没看源码时,我的理解是:    
1. Validator自己实现:BeanPostProcessor.   
2. 验证目标Object的方法/类上是否有注解(@Validated),如果有注解,则进行动态代理就好了,没想到,它的代码出乎我的意料.  

