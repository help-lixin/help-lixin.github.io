---
layout: post
title: 'Spring源码之SpringRunner(一)' 
date: 2021-09-06
author: 李新
tags:  Junit
---

### (1). 概述

前几天,稍微研究了下Junit,是因为IDEA(Eclipse)有插件,会回调Junit的单元测试,那么:Spring是如何与Junit进行整合的呢?  
答案就在:org.springframework.test.context.junit4.SpringRunner里.

### (2). SpringApplicationTest
> 查看简单的测试案例.   

```
// 1. RunWith会被IDEA回调执行
@RunWith(SpringRunner.class)
// 2. SpringRunner内部会通过反射获得当前类(SpringApplicationTest)的所有(注解)信息,并创建Web容器.
@SpringBootTest(classes = {HelloController.class, Conf.class})
public class SpringApplicationTest {

    @Autowired
    private IUserService userService;

    @Test
    public void testHello() {
        System.out.println("*************************testHello");
    }
}
```
### (3). SpringRunner
>  很简单,直接调用了父类:SpringJUnit4ClassRunner

```

// 1. 注意是final,所以,我们自己不能继承于,得继承于:SpringJUnit4ClassRunner
public final class SpringRunner 
			 // *****************************************************
			 // 2. SpringJUnit4ClassRunner
			 // *****************************************************
             extends SpringJUnit4ClassRunner {

	// 1. clazz = help.lixin.test.SpringApplicationTest
	public SpringRunner(Class<?> clazz) throws InitializationError {
		super(clazz);
	} // end 
}
```
### (4). SpringJUnit4ClassRunner
```
package org.springframework.test.context.junit4;

public class SpringJUnit4ClassRunner 
       // ***********************************************************
	   // 1. 在前面分析过:BlockJUnit4ClassRunner是Junit的核心类,会被IDEA或者Maven插件回调.
	   // ***********************************************************
       extends BlockJUnit4ClassRunner {
	
	private final TestContextManager testContextManager;
	
	// clazz = help.lixin.test.SpringApplicationTest
	public SpringJUnit4ClassRunner(Class<?> clazz) throws InitializationError {
		super(clazz);
		
		if (logger.isDebugEnabled()) {
			logger.debug("SpringJUnit4ClassRunner constructor called with [" + clazz + "]");
		}
		
		ensureSpringRulesAreNotPresent(clazz);
		
		// **********************************************************
		// 2. 创建:TestContextManager
		// **********************************************************
		this.testContextManager = createTestContextManager(clazz);
	} // end SpringJUnit4ClassRunner

	
	// 3. 创建:TestContextManager实例
	protected TestContextManager createTestContextManager(Class<?> clazz) {
		return new TestContextManager(clazz);
	} // end 	createTestContextManager
	
}	
```
### (5). TestContextManager
```
public class TestContextManager {
	
	private final TestContext testContext;
	
	
	public TestContextManager(Class<?> testClass) {
		this(
			// ******************************************************************
			// 2. 解析@BootstrapWith(SpringBootTestContextBootstrapper.class)
			// ******************************************************************
			BootstrapUtils.resolveTestContextBootstrapper(
			    // ******************************************************************
				// 1. 创建BootstrapContext
				// ******************************************************************
				BootstrapUtils.createBootstrapContext(testClass)
			)
		);
	}// end TestContextManager
	
	
	static BootstrapContext createBootstrapContext(Class<?> testClass) {
		// 1.1 创建:CacheAwareContextLoaderDelegate cacheAwareContextLoaderDelegate --> DefaultCacheAwareContextLoaderDelegate
		CacheAwareContextLoaderDelegate cacheAwareContextLoaderDelegate = createCacheAwareContextLoaderDelegate();
		Class<? extends BootstrapContext> clazz = null;
		try {
			// DEFAULT_BOOTSTRAP_CONTEXT_CLASS_NAME --> org.springframework.test.context.support.DefaultBootstrapContext
			clazz = (Class<? extends BootstrapContext>) ClassUtils.forName(DEFAULT_BOOTSTRAP_CONTEXT_CLASS_NAME, BootstrapUtils.class.getClassLoader());
			Constructor<? extends BootstrapContext> constructor = clazz.getConstructor(Class.class, CacheAwareContextLoaderDelegate.class);
			if (logger.isDebugEnabled()) {
				logger.debug(String.format("Instantiating BootstrapContext using constructor [%s]", constructor));
			}
			// 1.2 创建DefaultBootstrapContext对象,包裹着:被测试的类和DefaultCacheAwareContextLoaderDelegate
			return BeanUtils.instantiateClass(constructor, testClass, cacheAwareContextLoaderDelegate);
		}
		catch (Throwable ex) {
			throw new IllegalStateException("Could not load BootstrapContext [" + clazz + "]", ex);
		}
	}// end createBootstrapContext
	
	
	private static CacheAwareContextLoaderDelegate createCacheAwareContextLoaderDelegate() {
		Class<? extends CacheAwareContextLoaderDelegate> clazz = null;
		try {
			// DEFAULT_CACHE_AWARE_CONTEXT_LOADER_DELEGATE_CLASS_NAME --> org.springframework.test.context.cache.DefaultCacheAwareContextLoaderDelegate
			clazz = (Class<? extends CacheAwareContextLoaderDelegate>) ClassUtils.forName(DEFAULT_CACHE_AWARE_CONTEXT_LOADER_DELEGATE_CLASS_NAME, BootstrapUtils.class.getClassLoader());
			
			if (logger.isDebugEnabled()) {
				logger.debug(String.format("Instantiating CacheAwareContextLoaderDelegate from class [%s]",
					clazz.getName()));
			}
			
			// 1.1.1 创建:DefaultCacheAwareContextLoaderDelegate的实例
			return BeanUtils.instantiateClass(clazz, CacheAwareContextLoaderDelegate.class);
		}
		catch (Throwable ex) {
			throw new IllegalStateException("Could not load CacheAwareContextLoaderDelegate [" + clazz + "]", ex);
		}
	} // end createCacheAwareContextLoaderDelegate
	
	
	static TestContextBootstrapper resolveTestContextBootstrapper(BootstrapContext bootstrapContext) {
		Class<?> testClass = bootstrapContext.getTestClass();

		Class<?> clazz = null;
		try {
			// ******************************************************************
			// 3. 解析被测试类上的:@BootstrapWith(SpringBootTestContextBootstrapper.class)
			// ******************************************************************
			clazz = resolveExplicitTestContextBootstrapper(testClass);
			if (clazz == null) {
				// 4. 创建默认的Context
				clazz = resolveDefaultTestContextBootstrapper(testClass);
			}
			if (logger.isDebugEnabled()) {
				logger.debug(String.format("Instantiating TestContextBootstrapper for test class [%s] from class [%s]",
						testClass.getName(), clazz.getName()));
			}
			
			// ****************************************************************************
			// 5. 初始化,实际上可以理解为:Spring的容器了
			// org.springframework.boot.test.context.SpringBootTestContextBootstrapper
			// ****************************************************************************
			TestContextBootstrapper testContextBootstrapper =
					BeanUtils.instantiateClass(clazz, TestContextBootstrapper.class);
			testContextBootstrapper.setBootstrapContext(bootstrapContext);
			return testContextBootstrapper;
		} catch (IllegalStateException ex) {
			throw ex;
		} catch (Throwable ex) {
			throw new IllegalStateException("Could not load TestContextBootstrapper [" + clazz +
					"]. Specify @BootstrapWith's 'value' attribute or make the default bootstrapper class available.",
					ex);
		}
	} // end resolveTestContextBootstrapper
	
	
	// 判断类上是否有注解@WebAppConfiguration,如果有这个注解,则解析成:WebTestContextBootstrapper
	// 否则,解析成:DefaultTestContextBootstrapper
	private static Class<?> resolveDefaultTestContextBootstrapper(Class<?> testClass) throws Exception {
		ClassLoader classLoader = BootstrapUtils.class.getClassLoader();
		// WEB_APP_CONFIGURATION_ANNOTATION_CLASS_NAME --> org.springframework.test.context.web.WebAppConfiguration
		AnnotationAttributes attributes = AnnotatedElementUtils.findMergedAnnotationAttributes(testClass, WEB_APP_CONFIGURATION_ANNOTATION_CLASS_NAME, false, false);
		
		if (attributes != null) {
			// DEFAULT_WEB_TEST_CONTEXT_BOOTSTRAPPER_CLASS_NAME --> org.springframework.test.context.web.WebTestContextBootstrapper
			return ClassUtils.forName(DEFAULT_WEB_TEST_CONTEXT_BOOTSTRAPPER_CLASS_NAME, classLoader);
		}
		
		// DEFAULT_TEST_CONTEXT_BOOTSTRAPPER_CLASS_NAME --> org.springframework.test.context.support.DefaultTestContextBootstrapper
		return ClassUtils.forName(DEFAULT_TEST_CONTEXT_BOOTSTRAPPER_CLASS_NAME, classLoader);
	} // end resolveDefaultTestContextBootstrapper
}
```
### (6). SpringBootTestContextBootstrapper
> 我比较关心Properties的处理(如何让它与Apollo/Nacos整合),没想到的是Spring不会读取配置文件(application.properties),而是要求把配置文件附加在注解上    

```
// @SpringBootTest(classes = {HelloController.class, Conf.class}, properties = {"test.key=world"})

public class SpringBootTestContextBootstrapper extends DefaultTestContextBootstrapper {
	
	public TestContext buildTestContext() {
		TestContext context = super.buildTestContext();
		verifyConfiguration(context.getTestClass());
		WebEnvironment webEnvironment = getWebEnvironment(context.getTestClass());
		if (webEnvironment == WebEnvironment.MOCK
				&& deduceWebApplicationType() == WebApplicationType.SERVLET) {
			context.setAttribute(ACTIVATE_SERVLET_LISTENER, true);
		}
		else if (webEnvironment != null && webEnvironment.isEmbedded()) {
			context.setAttribute(ACTIVATE_SERVLET_LISTENER, false);
		}
		return context;
	}


	protected void processPropertySourceProperties(
				MergedContextConfiguration mergedConfig,
				List<String> propertySourceProperties) {
		Class<?> testClass = mergedConfig.getTestClass();
		// 读取注解上的配置
		// @SpringBootTest(classes = {HelloController.class, Conf.class}, properties = {"test.key=world"})
		String[] properties = getProperties(testClass);
		if (!ObjectUtils.isEmpty(properties)) {
			// 把所有的配置合并在一起
			propertySourceProperties.addAll(0, Arrays.asList(properties));
		}
		
		// 判断是否为:RANDOM_PORT
		if (getWebEnvironment(testClass) == WebEnvironment.RANDOM_PORT) {
			// 定义随机端口.
			propertySourceProperties.add("server.port=0");
		}
	}
}
```
### (7). 总结
Spring在单元测试方面还是比较用心的,只是,代码有一点点乱! 
