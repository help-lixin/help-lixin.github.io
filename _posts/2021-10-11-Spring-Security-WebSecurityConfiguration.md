---
layout: post
title: 'Spring Security源码之WebSecurityConfiguration(二)' 
date: 2021-10-11
author: 李新
tags:  SpringSecurity
---

### (1). 概述
```
在分析:DelegatingFilterProxy时,发现,内部又引用了一个Bean(springSecurityFilterChain),所以,要了解下:springSecurityFilterChain是在什么时候初始化的.                
我在分析SecurityAutoConfiguration时,只提了一嘴说:它导入了一堆的Bean,实际:springSecurityFilterChain的初始化就在这三个Import的Bean(SpringBootWebSecurityConfiguration/WebSecurityEnablerConfiguration/SecurityDataConfiguration)里定义的,只是定义比较深而已.            
```
### (2). WebSecurityEnablerConfiguration
不拉出一些没有用的代码了,其实,最终的关注点是在这个注解上@EnableWebSecurity,你会发现:这个注解又导入了其它的Bean,我们重点关注:WebSecurityConfiguration

```
@Retention(value = java.lang.annotation.RetentionPolicy.RUNTIME)
@Target(value = { java.lang.annotation.ElementType.TYPE })
@Documented
// *******************************************************************************
// 导入了其它的Bean(WebSecurityConfiguration/SpringWebMvcImportSelector)
// *******************************************************************************
@Import({ WebSecurityConfiguration.class, SpringWebMvcImportSelector.class })
@EnableGlobalAuthentication
@Configuration
public @interface EnableWebSecurity {
	boolean debug() default false;
}
```
### (3). WebSecurityConfiguration
> WebSecurityConfiguration的职责如下:   
+ 创建:AutowiredWebSecurityConfigurersIgnoreParents,通过它,可以获得,Spring中自定义的所有:WebSecurityConfigurer集合.   
+ 新创建一个WebSecurity,把上一步所有的WebSecurityConfigurer中的配置,设置到新创建的:WebSecurity上.  
+ 委托给WebSecurity去创建:springSecurityFilterChain(Filter),这个过程,放到下一小节,进行剖析.   

```
@Configuration
public class WebSecurityConfiguration implements ImportAware, BeanClassLoaderAware {
	private WebSecurity webSecurity;

	private Boolean debugEnabled;

	private List<SecurityConfigurer<Filter, WebSecurity>> webSecurityConfigurers;

	private ClassLoader beanClassLoader;

	@Autowired(required = false)
	private ObjectPostProcessor<Object> objectObjectPostProcessor;
	

    // ****************************************************************************************************
	//  1. AutowiredWebSecurityConfigurersIgnoreParents的作用在于:从Spring容器中,找出:WebSecurityConfigurer的实现.
	//  beanFactory.getBeansOfType(WebSecurityConfigurer.class);
	// ****************************************************************************************************
	@Bean
	public static AutowiredWebSecurityConfigurersIgnoreParents autowiredWebSecurityConfigurersIgnoreParents(
			ConfigurableListableBeanFactory beanFactory) {
		return new AutowiredWebSecurityConfigurersIgnoreParents(beanFactory);
	} // end AutowiredWebSecurityConfigurersIgnoreParents
	
	
	// ****************************************************************************************************
	// 2. setFilterChainProxySecurityConfigurer方法,用于获取所有的:WebSecurityConfigurer,并设置到:new出来的:WebSecurity对象上面.
	// ****************************************************************************************************
	@Autowired(required = false)
	public void setFilterChainProxySecurityConfigurer(
			// 
			ObjectPostProcessor<Object> objectPostProcessor,
			// ************************************************************************************
			// 3. 调用:autowiredWebSecurityConfigurersIgnoreParents.getWebSecurityConfigurers()方法
			//    获得所有的:SecurityConfigurer集合
			//    这也是为了什么,Spring Security支持配置:SecurityConfigurer
			// ************************************************************************************
			@Value("#{@autowiredWebSecurityConfigurersIgnoreParents.getWebSecurityConfigurers()}") List<SecurityConfigurer<Filter, WebSecurity>> webSecurityConfigurers) throws Exception {
		
		// *************************************************************************************
		// 4. new WebSecurity
		//    把WebSecurity交给:AutowireBeanFactoryObjectPostProcessor,目的在于,让WebSecurity支持Spring的生命周期(DisposableBean/SmartInitializingSingleton).
		// *************************************************************************************
		webSecurity = objectPostProcessor.postProcess(new WebSecurity(objectPostProcessor));
		if (debugEnabled != null) {
			webSecurity.debug(debugEnabled);
		}
		
		// 对List集合进行排序.
		Collections.sort(webSecurityConfigurers, AnnotationAwareOrderComparator.INSTANCE);
		
		
		Integer previousOrder = null;
		Object previousConfig = null;
		for (SecurityConfigurer<Filter, WebSecurity> config : webSecurityConfigurers) {
			Integer order = AnnotationAwareOrderComparator.lookupOrder(config);
			if (previousOrder != null && previousOrder.equals(order)) {
				throw new IllegalStateException(
						"@Order on WebSecurityConfigurers must be unique. Order of " + order + " was already used on " + previousConfig + ", so it cannot be used on " + config + " too.");
			}
			previousOrder = order;
			previousConfig = config;
		}
		
		// *************************************************************************************
		// 5. 遍历容器中所有的:SecurityConfigurer,并设置到刚new出来的:WebSecurity上
		// *************************************************************************************
		for (SecurityConfigurer<Filter, WebSecurity> webSecurityConfigurer : webSecurityConfigurers) {
			webSecurity.apply(webSecurityConfigurer);
		}
		this.webSecurityConfigurers = webSecurityConfigurers;
	} // end setFilterChainProxySecurityConfigurer
	
		
	// 6. 实例化一个事件监听器(不重要)	
	@Bean
	public static DelegatingApplicationListener delegatingApplicationListener() {
		return new DelegatingApplicationListener();
	} //end delegatingApplicationListener
	
    
	// *************************************************************************************
	// 7. 创建一个Bean(Filter),Bean名称为:springSecurityFilterChain
	// *************************************************************************************
	@Bean(name = AbstractSecurityWebApplicationInitializer.DEFAULT_FILTER_NAME)
	public Filter springSecurityFilterChain() throws Exception {
		boolean hasConfigurers = webSecurityConfigurers != null && !webSecurityConfigurers.isEmpty();
		if (!hasConfigurers) {
			WebSecurityConfigurerAdapter adapter = objectObjectPostProcessor.postProcess(new WebSecurityConfigurerAdapter() {});
			webSecurity.apply(adapter);
		}
		// *************************************************************************************
		// 通过:WebSecurity.build方法,构建出:springSecurityFilterChain=Filter
		//     springSecurityFilterChain是什么?有什么?留到下一小节去剖析.   
		// *************************************************************************************
		return webSecurity.build();
	}// end springSecurityFilterChain
}
```
### (4). 总结
SecurityConfigurer是Spring Secruity预留出来给我们进行配置的,最终会把SecurityConfigurer向核心的:WebSecurity靠扰,至于:WebSecurity是如何构建出:springSecurityFilterChain(Filter),留到下一小节来剖析.    