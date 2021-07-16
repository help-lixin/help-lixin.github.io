---
layout: post
title: 'Spring源码之:PropertySourceLocator' 
date: 2021-07-17
author: 李新
tags:  Spring 
---

### (1). 概述
今天在看Nacos的源码时,我发现:Nacos基于动态配置,并没有这样做:ConfigurableEnvironment.getPropertySources().addFirst(PropertySource),所以,特意写成一篇内容,记录下.   

### (2). 先看一个Spring的类(PropertySourceBootstrapConfiguration)
```
@Configuration
@EnableConfigurationProperties(PropertySourceBootstrapProperties.class)
public class PropertySourceBootstrapConfiguration 
		// 1. 很重要的实现,Spring在启动容器之前,会通过SPI,加载所有的实现类,并回调:initialize方法
       implements ApplicationContextInitializer<ConfigurableApplicationContext>, 
		Ordered {

	public static final String BOOTSTRAP_PROPERTY_SOURCE_NAME = BootstrapApplicationListener.BOOTSTRAP_PROPERTY_SOURCE_NAME+ "Properties";

	private static Log logger = LogFactory.getLog(PropertySourceBootstrapConfiguration.class);

	private int order = Ordered.HIGHEST_PRECEDENCE + 10;

    // ********************************************************************
	// 2. 注入在Spring容器里,所有PropertySourceLocator的实现类
	// ********************************************************************
	@Autowired(required = false)
	private List<PropertySourceLocator> propertySourceLocators = new ArrayList<>();
	
   public void initialize(ConfigurableApplicationContext applicationContext) {
	   // 3. 创建一个组合的:PropertySource
   		CompositePropertySource composite = new CompositePropertySource(BOOTSTRAP_PROPERTY_SOURCE_NAME);
		// 4. 对propertySourceLocators进行排序
   		AnnotationAwareOrderComparator.sort(this.propertySourceLocators);
   		boolean empty = true;
		
		// 5. 获得当前的上下文环境(ConfigurableEnvironment)
   		ConfigurableEnvironment environment = applicationContext.getEnvironment();
		
		// 6. 回调所有的:PropertySourceLocator.locate方法
   		for (PropertySourceLocator locator : this.propertySourceLocators) {
   			PropertySource<?> source = null;
   			source = locator.locate(environment);
   			if (source == null) {
   				continue;
   			}
			
   			logger.info("Located property source: " + source);
   			composite.addPropertySource(source);
   			empty = false;
   		}// end for
		
   		if (!empty) {
   			// 7. 从ConfigurableEnvironment中获得MutablePropertySources载体(它承载着所有的:PropertySources,即配置信息)
			MutablePropertySources propertySources = environment.getPropertySources();
			// 8. 获得配置项:${logging.config:}
			String logConfig = environment.resolvePlaceholders("${logging.config:}");
			// 日志处理不必太关心
			LogFile logFile = LogFile.get(environment);
			if (propertySources.contains(BOOTSTRAP_PROPERTY_SOURCE_NAME)) {
				propertySources.remove(BOOTSTRAP_PROPERTY_SOURCE_NAME);
			}
			
			// **********************************************************
			// 9. environment.getPropertySources().addLast(CompositePropertySource)
			// **********************************************************
			insertPropertySources(propertySources, composite);
			// ... ... 
   		}
   	} // end initialize
}	
```
### (3). 看一下PropertySourceLocator接口
```
public interface PropertySourceLocator {
	// 传入Environment,返回一个:PropertySource(Nacos自己实现了它)
	PropertySource<?> locate(Environment environment);
}
```
### (4). NacosPropertySourceLocator
```
@Order(0)
public class NacosPropertySourceLocator implements PropertySourceLocator {
	@Override
	public PropertySource<?> locate(Environment env) {
		// ... ...
		CompositePropertySource composite = new CompositePropertySource(NACOS_PROPERTY_SOURCE_NAME);
		// ... ...
		return composite;
	}
}
```
### (5). 总结
Spring在设计上,真的值得大家学习,它永远都会留出回调函数给大家使用(完全符合开闭原则),以防止你直接改它的源码.   