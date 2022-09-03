---
layout: post
title: 'Spring StateMachine源码StateMachineConfiguration(一)' 
date: 2022-09-03
author: 李新
tags:  SpringStateMachine
---

### (1). 概述
在前一小节,对Spring StateMachine有了一个简单的了解,在这一小节,主要对源码进行剖析,那么剖析源码的入口在哪呢? 

### (2). 源码入口
```
@EnableStateMachine(name="orderSingleMachine")
```
### (3). @EnableStateMachine
```
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@EnableAnnotationConfiguration
@Import({ 
	    StateMachineConfigurationImportSelector.class, 
		StateMachineCommonConfiguration.class, 
		// *******************************************************************
		// 状态机配置类
		// *******************************************************************
		StateMachineConfiguration.class,
		ObjectPostProcessorConfiguration.class })
@EnableWithStateMachine
public @interface EnableStateMachine {
	// 配置状态机的名称,最后与@WithStateMachine(name="orderSingleMachine")中配置的名称一样.
	String[] name() default {StateMachineSystemConstants.DEFAULT_ID_STATEMACHINE};

	boolean contextEvents() default true;
}
```
### (4). AbstractImportingAnnotationConfiguration(StateMachineConfiguration的父类)
```
public abstract class AbstractImportingAnnotationConfiguration<B extends AnnotationBuilder<O>, O> implements
		ImportBeanDefinitionRegistrar, BeanFactoryAware, EnvironmentAware {

	@Override
	public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
		// *****************************************************************
		// 获得注解(@EnableStateMachine(name="orderSingleMachine")),挂在哪个类上面(help.lixin.config.StateMachineConfig)
		// *****************************************************************
		List<Class<? extends Annotation>> annotationTypes = getAnnotations();
		Class<? extends Annotation> namedAnnotation = null;
		// *****************************************************************
		// 注解上的名称
		// *****************************************************************
		String[] names = null;
		ScopedProxyMode proxyMode = null;
		if (annotationTypes != null) {
			for (Class<? extends Annotation> annotationType : annotationTypes) {
				// 获得注解上的名称
				AnnotationAttributes attributes = AnnotationAttributes.fromMap(importingClassMetadata.getAnnotationAttributes(annotationType.getName(), false));
				if (attributes != null && attributes.containsKey("name")) {
					names = attributes.getStringArray("name");
					namedAnnotation = annotationType;
					break;
				}
			}
		}

        // ... ...

		BeanDefinition beanDefinition;
		try {
			// *****************************************************************
			// 委托给子类StateMachineConfiguration处理.
			// *****************************************************************
			beanDefinition = buildBeanDefinition(importingClassMetadata, namedAnnotation);
		} catch (Exception e) {
			throw new RuntimeException("Error with onConfigurers", e);
		}

		// implementation didn't return definition so don't continue registration
		if (beanDefinition == null) {
			return;
		}

		if (ObjectUtils.isEmpty(names)) {
			// ok, name(s) not given, generate one
			names = new String[] { beanNameGenerator.generateBeanName(beanDefinition, registry) };
		}

        // **************************************************************************************************
		// @EnableStateMachine(name="orderSingleMachine")
		// 向Spring容器中注册Bean,Bean名称为:orderSingleMachine,对应的Bean为:StateMachineDelegatingFactoryBean
		// **************************************************************************************************
		registry.registerBeanDefinition(names[0], beanDefinition);
		// ... ... 
	} // end registerBeanDefinitions
}
```
### (5). StateMachineConfiguration
```
@Configuration
public class StateMachineConfiguration<S, E> extends
		AbstractImportingAnnotationConfiguration<StateMachineConfigBuilder<S, E>, StateMachineConfig<S, E>> { 

	@Override
	protected BeanDefinition buildBeanDefinition(AnnotationMetadata importingClassMetadata, Class<? extends Annotation> namedAnnotation) throws Exception {

		String enableStateMachineEnclosingClassName = importingClassMetadata.getClassName();
		// for below classloader, see gh122
		Class<?> enableStateMachineEnclosingClass = ClassUtils.forName(enableStateMachineEnclosingClassName,ClassUtils.getDefaultClassLoader());
		// return null if it looks like @EnableStateMachine was annotated with class
		// not extending StateMachineConfigurer.
		if (!ClassUtils.isAssignable(StateMachineConfigurer.class, enableStateMachineEnclosingClass)) {
			return null;
		}

		// **************************************************************************************************
		// StateMachineDelegatingFactoryBean是状态机的装饰工厂.
		// **************************************************************************************************
		BeanDefinitionBuilder beanDefinitionBuilder = BeanDefinitionBuilder.rootBeanDefinition(StateMachineDelegatingFactoryBean.class);
		AnnotationAttributes esmAttributes = AnnotationAttributes.fromMap(importingClassMetadata.getAnnotationAttributes(EnableStateMachine.class.getName(), false));
		Boolean contextEvents = esmAttributes.getBoolean("contextEvents");

		// check if Scope annotation is defined and set scope from it
		AnnotationAttributes scopeAttributes = AnnotationAttributes.fromMap(importingClassMetadata.getAnnotationAttributes(Scope.class.getName(), false));
		if (scopeAttributes != null) {
			String scope = scopeAttributes.getString("value");
			if (StringUtils.hasText(scope)) {
				beanDefinitionBuilder.setScope(scope);
			}
		}
		
		// *******************************************************************
		// *******************************************************************
		// 为StateMachineDelegatingFactoryBean配置构建器:
		
		// 1. 创建一个空的状态机构造器.
		// StateMachineConfigBuilder<S, E> builder 
		
		// 2. 真实的状态机
		// Class<StateMachine<S, E>> clazz = StateMachine.class
		// String clazzName = help.lixin.StateMachineConfig
		
		 // 3. 是否开启上下文事件.
		 // Boolean contextEvents = true
		 
		beanDefinitionBuilder.addConstructorArgValue(builder);
		beanDefinitionBuilder.addConstructorArgValue(StateMachine.class);
		beanDefinitionBuilder.addConstructorArgValue(importingClassMetadata.getClassName());
		beanDefinitionBuilder.addConstructorArgValue(contextEvents);

		AbstractBeanDefinition beanDefinition = beanDefinitionBuilder.getBeanDefinition();

		// try to add more info about generics
		ResolvableType type = resolveFactoryObjectType(enableStateMachineEnclosingClass);
		if (type != null && beanDefinition instanceof RootBeanDefinition) {
			((RootBeanDefinition)beanDefinition).setTargetType(type);
		}
		return beanDefinition;
	} // end buildBeanDefinition
}
```
### (6). 总结
注解:@EnableStateMachine(name="orderSingleMachine")会向Spring容器中注册一个Bean,Bean的名称为:orderSingleMachine,类型为:StateMachineDelegatingFactoryBean.  