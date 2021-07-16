---
layout: post
title: 'Apollo源码学习之:Spring无缝整合(六)' 
date: 2021-07-16
author: 李新
tags:  Apollo
---

### (1). 概述
在前面查看了半天的:@EnableApolloConfig的源码,始终没有找到我们想要的内容.                                                     
即:Apollo向Spring靠拢,始终是要围绕着这个类:Config来做适配来着的,那咋办?                        
找/META-INF/spring.factories,这是Spring的启动之前的回调点.         

### (2). apollo-client-xxx.jar/META-INF/spring.factories
```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.ctrip.framework.apollo.spring.boot.ApolloAutoConfiguration
org.springframework.context.ApplicationContextInitializer=\
com.ctrip.framework.apollo.spring.boot.ApolloApplicationContextInitializer
org.springframework.boot.env.EnvironmentPostProcessor=\
com.ctrip.framework.apollo.spring.boot.ApolloApplicationContextInitializer
```
### (3). ApolloApplicationContextInitializer
```
public class ApolloApplicationContextInitializer 
       implements ApplicationContextInitializer<ConfigurableApplicationContext> , 
	              EnvironmentPostProcessor, 
				  Ordered {
	
	private static final String[] APOLLO_SYSTEM_PROPERTIES = {"app.id", ConfigConsts.APOLLO_CLUSTER_KEY, "apollo.cacheDir", "apollo.accesskey.secret", ConfigConsts.APOLLO_META_KEY, PropertiesFactory.APOLLO_PROPERTY_ORDER_ENABLE};
	
	private final ConfigPropertySourceFactory configPropertySourceFactory = SpringInjector.getInstance(ConfigPropertySourceFactory.class);
	
	// 1. Spring回调:EnvironmentPostProcessor.postProcessEnvironment
	public void postProcessEnvironment(ConfigurableEnvironment configurableEnvironment, SpringApplication springApplication) {
	    
		// 从ConfigurableEnvironment获取如下变量:
		// [app.id, apollo.cluster, apollo.cacheDir, apollo.accesskey.secret, apollo.meta, apollo.property.order.enable]
		// 重新设置到:  System.setProperty(propertyName, propertyValue);
		// should always initialize system properties like app.id in the first place
	    initializeSystemProperty(configurableEnvironment);
	
		// apollo.bootstrap.eagerLoad.enabled = false
	    Boolean eagerLoadEnabled = configurableEnvironment.getProperty(PropertySourcesConstants.APOLLO_BOOTSTRAP_EAGER_LOAD_ENABLED, Boolean.class, false);
		
		
	    //EnvironmentPostProcessor should not be triggered if you don't want Apollo Loading before Logging System Initialization
	    if (!eagerLoadEnabled) {
	      return;
	    }
	
		// apollo.bootstrap.enabled=true
	    Boolean bootstrapEnabled = configurableEnvironment.getProperty(PropertySourcesConstants.APOLLO_BOOTSTRAP_ENABLED, Boolean.class, false);
	
	    // bootstrapEnabled为true时,才会调用:initialize
	    if (bootstrapEnabled) {
	      initialize(configurableEnvironment);
	    }
	} // end postProcessEnvironment
	
	// 2. Spring回调:ApplicationContextInitializer.initialize
	public void initialize(ConfigurableApplicationContext context) {
	    ConfigurableEnvironment environment = context.getEnvironment();
		
		// apollo.bootstrap.enabled=true
	    if (!environment.getProperty(PropertySourcesConstants.APOLLO_BOOTSTRAP_ENABLED, Boolean.class, false)) {
	      logger.debug("Apollo bootstrap config is not enabled for context {}, see property: ${{}}", context, PropertySourcesConstants.APOLLO_BOOTSTRAP_ENABLED);
	      return;
	    }
		
	    logger.debug("Apollo bootstrap config is enabled for context {}", context);
		
	    initialize(environment);
	} // end initialize
	
	
	protected void initialize(ConfigurableEnvironment environment) {
		
		// 判断PropertySource配置里,是否有:ApolloBootstrapPropertySources
		// PropertySourcesConstants.APOLLO_BOOTSTRAP_PROPERTY_SOURCE_NAME = ApolloBootstrapPropertySources
		if (environment.getPropertySources().contains(PropertySourcesConstants.APOLLO_BOOTSTRAP_PROPERTY_SOURCE_NAME)) { //false
		  //already initialized
		  return;
		}
		
		// APOLLO_BOOTSTRAP_NAMESPACES = apollo.bootstrap.namespaces
		// NAMESPACE_APPLICATION = application
		// 获取配置文件中设置的namespaces: apollo.bootstrap.namespaces
		String namespaces = environment.getProperty(PropertySourcesConstants.APOLLO_BOOTSTRAP_NAMESPACES, ConfigConsts.NAMESPACE_APPLICATION);
		logger.debug("Apollo bootstrap namespaces: {}", namespaces);
		// 按逗号分隔namespaces
		List<String> namespaceList = NAMESPACE_SPLITTER.splitToList(namespaces);

		// 3. 创建一个组合的:CompositePropertySource
		CompositePropertySource composite = new CompositePropertySource(PropertySourcesConstants.APOLLO_BOOTSTRAP_PROPERTY_SOURCE_NAME);
		
		// 4. namespaceList = [TEST1.jdbc, application]
		for (String namespace : namespaceList) {
		  // ************************************************************************
		  // 终于见到核心类:Config
		  // ************************************************************************
		  Config config = ConfigService.getConfig(namespace);
		  
		  // ************************************************************************
		  // 通过ConfigPropertySourceFactory创建ConfigPropertySource
		  // ************************************************************************
		  composite.addPropertySource(configPropertySourceFactory.getConfigPropertySource(namespace, config));
		}
		
		// *********************************************************************
		// 向容器中注册:PropertySource,而且,还是放在首位,意味着:编写顺序在最后的namespace具有优先查找功能.
		// 比如: namespaces = [TEST1.jdbc, application]
		// 而 environment.getPropertySources().addFirst()后的结果是:  [application, TEST1.jdbc]
		//      key = server.port
		//     先在:application查找,找不到,再到:TEST1.jdbc里查找.
		// *********************************************************************
		environment.getPropertySources().addFirst(composite);
    }// end initialize
}
```
### (4). ConfigPropertySourceFactory
1) Apollo自定义了ConfigPropertySource(PropertySource的实现类).     
2) 同时,ConfigPropertySource内部持有Apollo的Config对象,最终会委托到:Config获取配置信息.   
3) 把ConfigPropertySource向Spring中注册(environment.getPropertySources().addFirst(composite)).   
4) 还有不懂的,[请参考我前面的文章,自定义PropertySource](https://www.lixin.help/2018/05/01/SpringCloud-RefreshScope.html)   

```
public class ConfigPropertySourceFactory {

  private final List<ConfigPropertySource> configPropertySources = Lists.newLinkedList();

  // ***************************************************************************
  // Config类提供了获取配置的方法: 
  //  String getProperty(String key, String defaultValue);
  // ***************************************************************************
  public ConfigPropertySource getConfigPropertySource(String name, Config source) {
	  // 1. 自定义:ConfigPropertySource属于:PropertySource的子类.
	  // 2. ConfigPropertySource内部又委托给了:Config类,到此,一切问题就已经打通了.
    ConfigPropertySource configPropertySource = new ConfigPropertySource(name, source);
	configPropertySources.add(configPropertySource);
    return configPropertySource;
  }

  public List<ConfigPropertySource> getAllConfigPropertySources() {
    return Lists.newLinkedList(configPropertySources);
  }
}
```
### (5). 总结
Spring与Apollo的结合,实际就是自定义了PropertySource,然后,向Spring的Environment中注册,获取配置最终会委托给:Config.getProperty方法,所以,后面,专心剖析:Config.  