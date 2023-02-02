---
layout: post
title: 'SpringCloud源码分析之SpringClientFactory' 
date: 2023-01-31
author: 李新
tags:  SpringCloud
---

### (1). 概述
两年前,在看Ribbon源码时,没有花时间记录源码,原因,赶时间,Ribbon其实并不简单,我为什么这么说呢?因为:Ribbon负载均衡是为每一个serviceId(微服务名称)生成一个:ApplicationContext,为什么要这样做?我们可以针对不同的微服务,配置不同的策略,如果配置在一起的话,就会相互干扰呢?

### (2). NamedContextFactory属性分析
```
public abstract class NamedContextFactory<C extends NamedContextFactory.Specification> implements 
		DisposableBean, 
		ApplicationContextAware {

	public interface Specification {
		String getName();

		Class<?>[] getConfiguration();
	}
    
	// ********************************************************************************************
	// key    : 是微服务名称
	// value  : AnnotationConfigApplicationContext
	// ********************************************************************************************
	private Map<String, AnnotationConfigApplicationContext> contexts = new ConcurrentHashMap<>();
	
	// ********************************************************************************************
	// key   : 微服务名称
	// value : 配置的Class(比如:com.alibaba.cloud.nacos.ribbon.NacosRibbonClientConfiguration)
	// ********************************************************************************************
	private Map<String, C> configurations = new ConcurrentHashMap<>();

	// ********************************************************************************************
	// 为容器配置父容器
	// ********************************************************************************************
	private ApplicationContext parent;
	
	// ********************************************************************************************
	// org.springframework.cloud.openfeign.FeignClientsConfiguration
	// ********************************************************************************************
	private Class<?> defaultConfigType;
	
	// ribbon
	private final String propertySourceName;
	// ribbon.client.name
	private final String propertyName;
}
```
### (3). NamedContextFactory.getContext
```
protected AnnotationConfigApplicationContext getContext(String name) {
	if (!this.contexts.containsKey(name)) {
		synchronized (this.contexts) {
			if (!this.contexts.containsKey(name)) {
				// ************************************************************
				// name为微服务的名称
				// ************************************************************
				this.contexts.put(name, createContext(name));
			}
		}
	}
	return this.contexts.get(name);
}
```
### (4). NamedContextFactory.createContext
```
protected AnnotationConfigApplicationContext createContext(String name) {
	// ***********************************************************************************
	// name : 为微服务称
	// 为每一个微服务创建一个ApplicationContext来着
	// ***********************************************************************************
	AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
	
	// 处理注解上配置的configuration
	// 比如:
	// name : test-provider
	// configuration : help.lixin.route.ribbon.config.EurekaRibbonClientConfig
	// @RibbonClients(defaultConfiguration=EurekaRibbonClientConfig.class)
	if (this.configurations.containsKey(name)) {
		for (Class<?> configuration : this.configurations.get(name).getConfiguration()) {
			// 这一步,实际是往Spring容器里添加配置对象(可以理解成Spring以前的一段XML即可)
			context.register(configuration);
		}
	}
	
	// 处理注解上配置的configuration
	// 比如:
	// name : default.help.lixin.route.ribbon.config.NacosRibbonClientConfig
	// configuration : help.lixin.route.ribbon.config.NacosRibbonClientConfig
	// @RibbonClients(defaultConfiguration=NacosRibbonClientConfig.class)
	for (Map.Entry<String, C> entry : this.configurations.entrySet()) {
		if (entry.getKey().startsWith("default.")) {
			for (Class<?> configuration : entry.getValue().getConfiguration()) {
				context.register(configuration);
			}
		}
	}
	// 上面的处理,实际就是往Spring容器里添加一堆的Bean定义信息
	
	// defaultConfigType = FeignClientsConfiguration
	// defaultConfigType = LoadBalancerClientConfiguration
	
	// 向Spring容器中注册多个Bean
	context.register(PropertyPlaceholderAutoConfiguration.class,this.defaultConfigType);
	
	// *********************************************************************************
	// 向Spring容器中注册配置信息,配置的名称不重要,重要的是:MapPropertySource里内部Hold住的Map
	// MapPropertySource = { 
	//    	ribbon.client.name : test-provider
	// }
	// *********************************************************************************
	context.getEnvironment().getPropertySources().addFirst(new MapPropertySource(
			this.propertySourceName,
			// propertyName = ribbon.client.name
			// name = test-provider(微服务的名称)
			Collections.<String, Object> singletonMap(this.propertyName, name)));
	
	if (this.parent != null) {
		// Uses Environment from parent as well as beans
		context.setParent(this.parent);
	}
	context.setDisplayName(generateDisplayName(name));
	context.refresh();
	return context;
}
```
### (5). RibbonClientConfiguration
> 对上面的内容好奇吗?为什么是往配置里加的是一条信息?

```
@Configuration
@EnableConfigurationProperties
@Import({HttpClientConfiguration.class, OkHttpRibbonConfiguration.class, RestClientRibbonConfiguration.class, HttpClientRibbonConfiguration.class})
public class RibbonClientConfiguration {
	// *****************************************************************
	// 重点就在于这里:会把服务名称(test-provider)注入到这里来
	// *****************************************************************
	@RibbonClientName
	private String name = "client";
	// ... ... 
}	
```
### (6). RibbonClientName注解
```
// *****************************************************************
// 与上面的遥相呼应. 
// *****************************************************************
@Value("${ribbon.client.name}")
public @interface RibbonClientName {
}
```
### (7). 总结
Spring的设计还是挺优雅的,通过不同上下文可以做到完全隔离,但是,也增加了复杂度,一般开发人员估计难理解. 