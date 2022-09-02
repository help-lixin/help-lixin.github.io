---
layout: post
title: 'Spring Plugin源码之EnablePluginRegistries(一)' 
date: 2022-09-02
author: 李新
tags:  SpringPlugin
---

### (1). 概述
前面对Spring Plugin有了一个简单的使用,在这里对Spring Plugin进行源码剖析.

### (2). 入口在哪?
```
@EnablePluginRegistries({ SamplePlugin.class, AnotherPlugin.class })
```
### (3). EnablePluginRegistries
```
package org.springframework.plugin.core.config;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Inherited;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.annotation.Import;
import org.springframework.plugin.core.Plugin;
import org.springframework.plugin.core.PluginRegistry;


@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
// ***************************************************************
// 重点:通过@Import导入了一个ImportBeanDefinitionRegistrar的实现
// ***************************************************************
@Import(PluginRegistriesBeanDefinitionRegistrar.class)
public @interface EnablePluginRegistries {

	Class<? extends Plugin<?>>[] value();
}
```
### (4). PluginRegistriesBeanDefinitionRegistrar
```
public class PluginRegistriesBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {
	
	
	public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
		// 从容器中,获得标注有注解:@EnablePluginRegistries的类
		Map<String, Object> annotationAttributes = importingClassMetadata.getAnnotationAttributes(EnablePluginRegistries.class.getName());

		if (annotationAttributes == null) {
			LOG.info("No EnablePluginRegistries annotation found on type {}!", importingClassMetadata.getClassName());
			return;
		}

        // 获得注解@EnablePluginRegistries(value={ SamplePlugin.class, AnotherPlugin.class })
		Class<?>[] types = (Class<?>[]) annotationAttributes.get("value");

		
		for (Class<?> type : types) {
			// ***********************************************************
			// 定义Bean信息(PluginRegistryFactoryBean)
			// ***********************************************************
			RootBeanDefinition beanDefinition = new RootBeanDefinition(getTargetType(type, PluginRegistryFactoryBean.class));
			beanDefinition.setTargetType(getTargetType(type, OrderAwarePluginRegistry.class));
			beanDefinition.setBeanClass(PluginRegistryFactoryBean.class);
			// ***********************************************************
			// 配置type为Plugin的实现类(SamplePlugin/AnotherPlugin)
			// ***********************************************************
			beanDefinition.getPropertyValues().addPropertyValue("type", type);
			
			// 看下Plugin接口上是否拥有注解@Qualifier
			Qualifier annotation = type.getAnnotation(Qualifier.class);

			// If the plugin interface has a Qualifier annotation, propagate that to the bean definition of the registry
			if (annotation != null) {
				AutowireCandidateQualifier qualifierMetadata = new AutowireCandidateQualifier(Qualifier.class);
				qualifierMetadata.setAttribute(AutowireCandidateQualifier.VALUE_KEY, annotation.value());
				beanDefinition.addQualifier(qualifierMetadata);
			}

			// Default
			String beanName = annotation == null //
					? StringUtils.uncapitalize(type.getSimpleName() + "Registry") // SamplePluginRegistry
					: annotation.value();  // 使用注解(@Qualifier)上的名称
			// **************************************************
			// 向Spring中注册FactoryBean(PluginRegistryFactoryBean)
			// **************************************************
			registry.registerBeanDefinition(beanName, beanDefinition);
		} // end for
	} // end registerBeanDefinitions
	
}	
```
### (5). 总结
注解@EnablePluginRegistries的目的是向Spring容器中注册Bean(PluginRegistryFactoryBean)
