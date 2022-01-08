---
layout: post
title: 'Spring Data Elasticsearch源码之RepositoryConfigurationDelegate(五)' 
date: 2022-01-02
author: 李新
tags:  Spring-Data-Elasticsearch
---

### (1). 概述
在上一篇,分析了到:ElasticsearchRepositoriesAutoConfiguration会运用@Import(ElasticsearchRepositoriesRegistrar.class)导入Bean.    
ElasticsearchRepositoriesRegistrar的主要目的是向Spring容器中注册Bean,最终又委托给了:RepositoryConfigurationDelegate,在这一小篇主要分析:RepositoryConfigurationDelegate.   

### (2). RepositoryConfigurationDelegate
```
public List<BeanComponentDefinition> registerRepositoriesIn(BeanDefinitionRegistry registry,
			// *******************************************************************
			//  1. ElasticsearchRepositoryConfigExtension
			// *******************************************************************
			RepositoryConfigurationExtension extension) {

	if (logger.isInfoEnabled()) {
		logger.info(LogMessage.format("Bootstrapping Spring Data %s repositories in %s mode.", extension.getModuleName(), configurationSource.getBootstrapMode().name()));
	}
	
	extension.registerBeansForRoot(registry, configurationSource);

	RepositoryBeanDefinitionBuilder builder = new RepositoryBeanDefinitionBuilder(registry, extension, configurationSource, resourceLoader, environment);
	List<BeanComponentDefinition> definitions = new ArrayList<>();

	StopWatch watch = new StopWatch();

	if (logger.isDebugEnabled()) {
		logger.debug(LogMessage.format("Scanning for %s repositories in packages %s.", //
				extension.getModuleName(), //
				configurationSource.getBasePackages().stream().collect(Collectors.joining(", "))));
	}

	watch.start();

	// *********************************************************************************
	// 2.委托给:ElasticsearchRepositoryConfigExtension获得所有:ElasticsearchRepository接口的实现接口
	// *********************************************************************************
	Collection<RepositoryConfiguration<RepositoryConfigurationSource>> configurations = extension.getRepositoryConfigurations(configurationSource, resourceLoader, inMultiStoreMode);

	Map<String, RepositoryConfiguration<?>> configurationsByRepositoryName = new HashMap<>(configurations.size());
	for (RepositoryConfiguration<? extends RepositoryConfigurationSource> configuration : configurations) {
		configurationsByRepositoryName.put(configuration.getRepositoryInterface(), configuration);
		
		// *********************************************************************************
		// 3. 构建:BeanDefinitionBuilder
		//     com.gerp.listing.ebay.dao.IBookDao对应的工厂类:
		//     org.springframework.data.elasticsearch.repository.support.ElasticsearchRepositoryFactoryBean
		// *********************************************************************************
		BeanDefinitionBuilder definitionBuilder = builder.build(configuration);
		
		// 后置回调处理
		extension.postProcess(definitionBuilder, configurationSource);
		
		if (isXml) {
			extension.postProcess(definitionBuilder, (XmlRepositoryConfigurationSource) configurationSource);
		} else {
			extension.postProcess(definitionBuilder, (AnnotationRepositoryConfigurationSource) configurationSource);
		}
		
		
		AbstractBeanDefinition beanDefinition = definitionBuilder.getBeanDefinition();
		beanDefinition.setResourceDescription(configuration.getResourceDescription());
		String beanName = configurationSource.generateBeanName(beanDefinition);
		if (logger.isTraceEnabled()) {
			logger.trace(LogMessage.format(REPOSITORY_REGISTRATION, extension.getModuleName(), beanName, configuration.getRepositoryInterface(), configuration.getRepositoryFactoryBeanClassName()));
		}
		
		// help.lixin.IBookDao
		beanDefinition.setAttribute(FACTORY_BEAN_OBJECT_TYPE, configuration.getRepositoryInterface());
		
		// 向Spring容器中,注册Bean
		registry.registerBeanDefinition(beanName, beanDefinition);
		definitions.add(new BeanComponentDefinition(beanDefinition, beanName));
	}

	potentiallyLazifyRepositories(configurationsByRepositoryName, registry, configurationSource.getBootstrapMode());

	watch.stop();

	if (logger.isInfoEnabled()) {
		logger.info(LogMessage.format("Finished Spring Data repository scanning in %s ms. Found %s %s repository interfaces.", watch.getLastTaskTimeMillis(), configurations.size(), extension.getModuleName()));
	}
	
	return definitions;
}
```
### (3). 总结
RepositoryConfigurationDelegate最主要目的是向Spring中注册BeanFactory(help.lixin.IBookDao --> ElasticsearchRepositoryFactoryBean).   

