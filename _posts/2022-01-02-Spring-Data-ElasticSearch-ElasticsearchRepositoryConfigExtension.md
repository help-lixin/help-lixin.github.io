---
layout: post
title: 'Spring Data Elasticsearch源码之ElasticsearchRepositoryConfigExtension(六)' 
date: 2022-01-02
author: 李新
tags:  Spring-Data-Elasticsearch
---

### (1). 概述
前面剖析了RepositoryConfigurationDelegate,它主要是向Spring容器中注册Bean,但是,注册哪些Bean呢?却又委托给了:ElasticsearchRepositoryConfigExtension

### (2). RepositoryConfigurationExtensionSupport
```
public <T extends RepositoryConfigurationSource> Collection<RepositoryConfiguration<T>> getRepositoryConfigurations(
			T configSource, ResourceLoader loader, boolean strictMatchesOnly) {
	Assert.notNull(configSource, "ConfigSource must not be null!");
	Assert.notNull(loader, "Loader must not be null!");

	Set<RepositoryConfiguration<T>> result = new HashSet<>();
	
	// ********************************************************************************
	// 委托给:RepositoryConfigurationSourceSupport进行类的扫描,此处不再详细剖析下去了
	// ********************************************************************************
	for (BeanDefinition candidate : configSource.getCandidates(loader)) {
		
		RepositoryConfiguration<T> configuration = getRepositoryConfiguration(candidate, configSource);
		Class<?> repositoryInterface = loadRepositoryInterface(configuration,
				getConfigurationInspectionClassLoader(loader));

		if (repositoryInterface == null) {
			result.add(configuration);
			continue;
		}
		
		RepositoryMetadata metadata = AbstractRepositoryMetadata.getMetadata(repositoryInterface);
		
		// ********************************************************************************
		// 2. isStrictRepositoryCandidate验证class是否为:ElasticsearchRepository
		// ********************************************************************************
		boolean qualifiedForImplementation = !strictMatchesOnly || configSource.usesExplicitFilters() || isStrictRepositoryCandidate(metadata);
		
		if (qualifiedForImplementation && useRepositoryConfiguration(metadata)) {
			result.add(configuration);
		}
	}

	return result;
} // end getRepositoryConfigurations


protected boolean isStrictRepositoryCandidate(RepositoryMetadata metadata) {
	// ... ... 
	// ********************************************************************************
	// 3. 获得支持的类型
	// ********************************************************************************
	Collection<Class<?>> types = getIdentifyingTypes();
	Collection<Class<? extends Annotation>> annotations = getIdentifyingAnnotations();
	String moduleName = getModuleName();
	
	if (types.isEmpty() && annotations.isEmpty()) {
		if (!noMultiStoreSupport) {
			logger.warn(LogMessage.format("Spring Data %s does not support multi-store setups!", moduleName));
			noMultiStoreSupport = true;
			return false;
		}
	}
	
	
	// help.lixin.dao.IBookDAO 
	Class<?> repositoryInterface = metadata.getRepositoryInterface();
	
	for (Class<?> type : types) {
		// ********************************************************************************
		// 5. 判断(type = ElasticsearchRepository) 
		// ********************************************************************************
		if (type.isAssignableFrom(repositoryInterface)) { 
			return true;
		}
	}
	
	// ... ... 
	return false;
} 
// end isStrictRepositoryCandidate
 

protected Collection<Class<?>> getIdentifyingTypes() {
	// ********************************************************************************
	// 4. 支持的类型
	// ********************************************************************************
	return Arrays.asList(ElasticsearchRepository.class, ElasticsearchRepository.class);
}
```
### (3). 总结
ElasticsearchRepositoryConfigExtension最主要的目的是扫描class,并判断是否为:ElasticsearchRepository类型,是的话,则添加到Spring容器中.  