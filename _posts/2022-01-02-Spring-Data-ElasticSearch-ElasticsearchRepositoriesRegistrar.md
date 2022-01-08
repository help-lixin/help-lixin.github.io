---
layout: post
title: 'Spring Data Elasticsearch源码之ElasticsearchRepositoriesRegistrar(四)' 
date: 2022-01-02
author: 李新
tags:  Spring-Data-Elasticsearch
---

### (1). 概述
前面有对Spring Data Elasticsearch有了一个比较简单的入门,是否好奇,为什么写一个接口(BookRepository),实现ElasticsearchRepository,即可,拥有基本的增删改查操作,Spring Data Elasticsearch底层到底做了什么?     
底层原理就在:ElasticsearchRepositoriesRegistrar里.

### (2). ElasticsearchRepositoriesAutoConfiguration
```
package org.springframework.boot.autoconfigure.data.elasticsearch;

@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({ Client.class, ElasticsearchRepository.class })
@ConditionalOnProperty(prefix = "spring.data.elasticsearch.repositories", name = "enabled", havingValue = "true",
		matchIfMissing = true)
@ConditionalOnMissingBean(ElasticsearchRepositoryFactoryBean.class)
// ******************************************************************************************
// 1. @import相当于导入一个配置类
// ******************************************************************************************
@Import(ElasticsearchRepositoriesRegistrar.class)
public class ElasticsearchRepositoriesAutoConfiguration {
}
```
### (3). AbstractRepositoryConfigurationSourceSupport
```
package org.springframework.boot.autoconfigure.data;

public abstract class AbstractRepositoryConfigurationSourceSupport 
    // ********************************************************************
	// 1. 实现了Spring的方法:ImportBeanDefinitionRegistrar
	// ********************************************************************
    implements ImportBeanDefinitionRegistrar, BeanFactoryAware, ResourceLoaderAware, EnvironmentAware {

	private ResourceLoader resourceLoader;	
	private BeanFactory beanFactory;
	private Environment environment;

    // ********************************************************************
	// 2. Spring会回调访方法
	// ********************************************************************
	@Override
	public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
		// 委托给了另一个方法.
		registerBeanDefinitions(importingClassMetadata, registry, null);
	} // end registerBeanDefinitions

    // ********************************************************************
	// 3. 注册Bean定义信息
	// ********************************************************************
	@Override
	public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry, BeanNameGenerator importBeanNameGenerator) {
		RepositoryConfigurationDelegate delegate = new RepositoryConfigurationDelegate(
		// *************************************************************************
		// 4. 获得配置类(ElasticsearchRepositoriesRegistrar$EnableElasticsearchRepositoriesConfiguration)
		// *************************************************************************
		getConfigurationSource(registry, importBeanNameGenerator), this.resourceLoader, this.environment);
		
		// *************************************************************************
		// 5. 委托给:RepositoryConfigurationDelegate进行Bean的注册
		// getRepositoryConfigurationExtension() == ElasticsearchRepositoryConfigExtension
		// *************************************************************************
		delegate.registerRepositoriesIn(registry, getRepositoryConfigurationExtension());
	} // end registerBeanDefinitions
	
	// *************************************************************************
	// 4.1 获得(ElasticsearchRepositoriesRegistrar$EnableElasticsearchRepositoriesConfiguration)类上的注解(@@EnableElasticsearchRepositories)
	// *************************************************************************
	private AnnotationRepositoryConfigurationSource getConfigurationSource(BeanDefinitionRegistry registry, BeanNameGenerator importBeanNameGenerator) {
		AnnotationMetadata metadata = AnnotationMetadata.introspect(getConfiguration());
		return new AutoConfiguredAnnotationRepositoryConfigurationSource(metadata, getAnnotation(), this.resourceLoader, this.environment, registry, importBeanNameGenerator) {
		};
	} // end getConfigurationSource
	
}
```
### (4). ElasticsearchRepositoriesRegistrar
```
package org.springframework.boot.autoconfigure.data.elasticsearch;

class ElasticsearchRepositoriesRegistrar extends AbstractRepositoryConfigurationSourceSupport {
	
	// 定义注解
	@Override
	protected Class<? extends Annotation> getAnnotation() {
		return EnableElasticsearchRepositories.class;
	}

	// 配置类
	@Override
	protected Class<?> getConfiguration() {
		return EnableElasticsearchRepositoriesConfiguration.class;
	}

	// 2. 定义配置扩展
	@Override
	protected RepositoryConfigurationExtension getRepositoryConfigurationExtension() {
		return new ElasticsearchRepositoryConfigExtension();
	}

	// 1. 定义注解
	@EnableElasticsearchRepositories
	private static class EnableElasticsearchRepositoriesConfiguration {
	}

}
```
### (5). 总结
Spring Data Elasticsearch在启动时,加载类ElasticsearchRepositoriesRegistrar$EnableElasticsearchRepositoriesConfiguration以及注解:@EnableElasticsearchRepositories,最终会委托给:RepositoryConfigurationDelegate进行Bean的注册.   