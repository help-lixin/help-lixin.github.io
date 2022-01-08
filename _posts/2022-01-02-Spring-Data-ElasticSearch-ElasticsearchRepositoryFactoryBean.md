---
layout: post
title: 'Spring Data Elasticsearch源码之ElasticsearchRepositoryFactoryBean(七)' 
date: 2022-01-02
author: 李新
tags:  Spring-Data-Elasticsearch
---

### (1). 概述
在这一篇,主要剖析ElasticsearchRepository的具体实现.  

### (2). RepositoryFactoryBeanSupport
```
public void afterPropertiesSet() {
	this.factory = createRepositoryFactory();
	this.factory.setQueryLookupStrategyKey(queryLookupStrategyKey);
	this.factory.setNamedQueries(namedQueries);
	this.factory.setEvaluationContextProvider(evaluationContextProvider.orElseGet(() -> QueryMethodEvaluationContextProvider.DEFAULT));
	this.factory.setBeanClassLoader(classLoader);
	this.factory.setBeanFactory(beanFactory);

	if (publisher != null) {
		this.factory.addRepositoryProxyPostProcessor(new EventPublishingRepositoryProxyPostProcessor(publisher));
	}

	repositoryBaseClass.ifPresent(this.factory::setRepositoryBaseClass);

	this.repositoryFactoryCustomizers.forEach(customizer -> customizer.customize(this.factory));

	RepositoryFragments customImplementationFragment = customImplementation //
			.map(RepositoryFragments::just) //
			.orElseGet(RepositoryFragments::empty);

	RepositoryFragments repositoryFragmentsToUse = this.repositoryFragments //
			.orElseGet(RepositoryFragments::empty) //
			.append(customImplementationFragment);
    
	this.repositoryMetadata = this.factory.getRepositoryMetadata(repositoryInterface);

	// Make sure the aggregate root type is present in the MappingContext (e.g. for auditing)
	this.mappingContext.ifPresent(it -> it.getPersistentEntity(repositoryMetadata.getDomainType()));
	
	// *************************************************************************************
	// 1. 委托给ElasticsearchRepositoryFactory创建:Repository
	// *************************************************************************************
	this.repository = Lazy.of(() -> this.factory.getRepository(repositoryInterface, repositoryFragmentsToUse));

	if (!lazyInit) { 
		this.repository.get();
	} // end 
}
```
### (3). RepositoryFactorySupport
```
public <T> T getRepository(Class<T> repositoryInterface, RepositoryFragments fragments) {

	if (logger.isDebugEnabled()) {
		logger.debug(LogMessage.format("Initializing repository instance for %s…", repositoryInterface.getName()));
	}

	Assert.notNull(repositoryInterface, "Repository interface must not be null!");
	Assert.notNull(fragments, "RepositoryFragments must not be null!");

	RepositoryMetadata metadata = getRepositoryMetadata(repositoryInterface);
	RepositoryComposition composition = getRepositoryComposition(metadata, fragments);
	RepositoryInformation information = getRepositoryInformation(metadata, composition);

	validate(information, composition);
	
	
	// target = SimpleElasticsearchRepository
	Object target = getTargetRepository(information);
	
	// *************************************************************************************
	// 2. 根据接口创建动态代理.
	// *************************************************************************************
	// Create proxy
	ProxyFactory result = new ProxyFactory();
	// target = SimpleElasticsearchRepository
	result.setTarget(target);
	result.setInterfaces(repositoryInterface, Repository.class, TransactionalProxy.class);

	if (MethodInvocationValidator.supports(repositoryInterface)) {
		result.addAdvice(new MethodInvocationValidator());
	}

	result.addAdvisor(ExposeInvocationInterceptor.ADVISOR);

	postProcessors.forEach(processor -> processor.postProcess(result, information));

	if (DefaultMethodInvokingMethodInterceptor.hasDefaultMethods(repositoryInterface)) {
		result.addAdvice(new DefaultMethodInvokingMethodInterceptor());
	}

	ProjectionFactory projectionFactory = getProjectionFactory(classLoader, beanFactory);
	Optional<QueryLookupStrategy> queryLookupStrategy = getQueryLookupStrategy(queryLookupStrategyKey,
			evaluationContextProvider);
	result.addAdvice(new QueryExecutorMethodInterceptor(information, projectionFactory, queryLookupStrategy,
			namedQueries, queryPostProcessors, methodInvocationListeners));

	composition = composition.append(RepositoryFragment.implemented(target));
	result.addAdvice(new ImplementationMethodExecutionInterceptor(information, composition, methodInvocationListeners));
	
	// *************************************************************************************
	// 3. 创建动态代理之后的对象
	// *************************************************************************************
	T repository = (T) result.getProxy(classLoader);

	if (logger.isDebugEnabled()) {
		logger
				.debug(LogMessage.format("Finished creation of repository instance for {}.", repositoryInterface.getName()));
	}
	
	// 返回动态代理的对象
	return repository;
}
```
### (4). 总结
从上面的分析结果能看出,ElasticsearchRepository接口的所有操作,全都会委派给SimpleElasticsearchRepository.  

### (5). 架构图如下
!["架构图如下"](/assets/spring-data-elasticsearch/imgs/SpringDataElasticSearch.jpg)  