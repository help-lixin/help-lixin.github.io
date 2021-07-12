---
layout: post
title: 'Spring Cache源码剖析(三)'
date: 2021-07-05
author: 李新
tags:  Spring 
---

### (1). 前言
在前面对Spring+Redis有了一个简单的案例入门,在这里将对Spring Cache部份的源码进行剖析.  

### (2). Spring Cache类图
> CacheManager固名思义:就是对Cache进行管理,可以管理多个Cache.         
> Cache是对缓存的一些基本操作(添加数据到缓存/剔除缓存中的数据/清空整个缓存...)         
> CacheResolver在AOP时会需要根据注解,解析出Cache对象.        
> KeyGenerator用于缓存时自定义KEY策略,注意,与注解上的key是互斥的.         

!["CacheManager"](/assets/spring/imgs/spring_CacheManager.jpg)    
!["Cache"](/assets/spring/imgs/spring_Cache.jpg)    
!["CacheResolver"](/assets/spring/imgs/spring_CacheResolver.jpg)   
!["KeyGenerator"](/assets/spring/imgs/spring_KeyGenerator.jpg)   

### (3). Spring Cache是如何进行AOP拦截的?
Spring Cache的入口程序就在注解上:@EnableCaching.  

```
// 相当于spring xml中的<import resource="xxxx.xml"/>
// 1. 导入相关的bean
@Import(CachingConfigurationSelector.class)
public @interface EnableCaching {
}// end EnableCaching


public class CachingConfigurationSelector 
       extends AdviceModeImportSelector<EnableCaching> {
    
	// ... ...
	// 2. Spring会回调该方法.
    public String[] selectImports(AdviceMode adviceMode) {
		switch (adviceMode) {
			case PROXY:  
				return getProxyImports();
			case ASPECTJ:
				return getAspectJImports();
			default:
				return null;
		}
	}// end
	
	// 3. 向Spring中注册bean
	private String[] getProxyImports() {
		List<String> result = new ArrayList<>(3);
		result.add(AutoProxyRegistrar.class.getName());
		// ************************************************************
		// 这是重点,向Spring中注册了一个Bean
		// ************************************************************
		result.add(ProxyCachingConfiguration.class.getName());
		if (jsr107Present && jcacheImplPresent) {
			result.add(PROXY_JCACHE_CONFIGURATION_CLASS);
		}
		return StringUtils.toStringArray(result);
	}// end getProxyImports
	
	// ... ...
}// end CachingConfigurationSelector
```

### (4). ProxyCachingConfiguration的作用是什么
> 1. ProxyCachingConfiguration是一个配置类(@Configuration).   
> 2. 创建CacheInterceptor,它属于AOP进行动态代理的核心.    
> 3. 创建BeanFactoryCacheOperationSourceAdvisor,这个属于定义要拦截的point.  

```
@Configuration
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
public class ProxyCachingConfiguration extends AbstractCachingConfiguration {

	@Bean(name = CacheManagementConfigUtils.CACHE_ADVISOR_BEAN_NAME)
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public BeanFactoryCacheOperationSourceAdvisor cacheAdvisor() {
		// 配置切面
		BeanFactoryCacheOperationSourceAdvisor advisor = new BeanFactoryCacheOperationSourceAdvisor();
		// 设置point
		advisor.setCacheOperationSource(cacheOperationSource());
		// 切面的逻辑代码
		advisor.setAdvice(cacheInterceptor());
		if (this.enableCaching != null) {
			advisor.setOrder(this.enableCaching.<Integer>getNumber("order"));
		}
		return advisor;
	}

	// 对方法上的注解进行解析,并返回方法上的注解信息.
	// Collection<CacheOperation> getCacheOperations(Method method, @Nullable Class<?> targetClass);
	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public CacheOperationSource cacheOperationSource() {
		return new AnnotationCacheOperationSource();
	}

    // ******************************************************************
	// CacheInterceptor类是重点,业务代码请求时,如果配置缓存注解,都会进入该类的invoke方法.
	// ******************************************************************
	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public CacheInterceptor cacheInterceptor() {
		CacheInterceptor interceptor = new CacheInterceptor();
		interceptor.configure(this.errorHandler, this.keyGenerator, this.cacheResolver, this.cacheManager);
		interceptor.setCacheOperationSource(cacheOperationSource());
		return interceptor;
	}
}
```
### (5). CacheInterceptor初始化做了什么?
> 1. 创建:CacheResolver,固名思义:这个类就是读取注解上的内容(转换成:CacheOperationInvocationContext),并获得真实的Cache.        
> 2. 从Spring容器中(BeanFactory),获得CacheManager,并Hold住.       
> 所以,你只要自己实现:CacheManager,并交给Spring即可(切记,当有多个实例的时候,要配一个@Primary),从代码上来看,它是不支持多个的.  

```
public class CacheInterceptor
		// *********************************************************************
		// 1. 继承父类CacheAspectSupport
		// *********************************************************************
       extends CacheAspectSupport 
	   implements MethodInterceptor, Serializable {
}// end CacheInterceptor


public abstract class CacheAspectSupport 
        extends AbstractCacheInvoker
		implements BeanFactoryAware, 
		InitializingBean, 
		// **************************************************************
		// 2. 实现了spring的org.springframework.beans.factory.SmartInitializingSingleton类
		//    Spring初始化完之后,会调用:SmartInitializingSingleton.afterSingletonsInstantiated
		// **************************************************************
		SmartInitializingSingleton {
	
	private CacheOperationSource cacheOperationSource;
	
	private SingletonSupplier<KeyGenerator> keyGenerator = SingletonSupplier.of(SimpleKeyGenerator::new);

	private SingletonSupplier<CacheResolver> cacheResolver;
	
	private BeanFactory beanFactory;
	
	private boolean initialized = false;
	
	
	public void afterSingletonsInstantiated() {
		if (getCacheResolver() == null) {
			// ... ...
			// **********************************************************************
			// 3. 从Spring容器中,获得:CacheManager,并Hold住.
			// **********************************************************************
			setCacheManager(this.beanFactory.getBean(CacheManager.class));
			// ... ... 
		}
		this.initialized = true;
	}// end afterSingletonsInstantiated
	
	public void setCacheManager(CacheManager cacheManager) {
		// *********************************************************
		// 4. 调用setCacheManager时,会通过:SimpleCacheResolver包裹着:CacheManager
		// 所以,也能猜得到了:CacheInterceptor内部,会调用:SimpleCacheResolver驱动CacheManager了.
		// *********************************************************
		this.cacheResolver = SingletonSupplier.of(new SimpleCacheResolver(cacheManager));
	}// end setCacheManager
	
}// end CacheAspectSupport
```
### (6). Spring Cache 如何避免缓存穿透
> @Cacheable(cacheNames = "users" , sync = true),当对相同的key进行操作时,会运用Java的synchronized关键字,在同一个进程里,只会允许一个线程去访问后面的业务代码.  

```
private Object execute(final CacheOperationInvoker invoker, Method method, CacheOperationContexts contexts) {
	// 在注解上是否开启了sync=true
	if (contexts.isSynchronized()) {
		CacheOperationContext context = contexts.get(CacheableOperation.class).iterator().next();
		if (isConditionPassing(context, CacheOperationExpressionEvaluator.NO_RESULT)) {
			Object key = generateKey(context, CacheOperationExpressionEvaluator.NO_RESULT);
			Cache cache = context.getCaches().iterator().next();
			try {
				return wrapCacheValue(
				      method, 
					  // *************************************************************************
					  // 看方法签名,get方法需要一个回调函数,这个回调函数,才会真正的触发业务代码的调用.
					  // cache.get(Object key, Callable<T> valueLoader)
					  // *************************************************************************
				      cache.get(key, () -> unwrapReturnValue(invokeOperation(invoker)))
				);
			}
			catch (Cache.ValueRetrievalException ex) {
				throw (CacheOperationInvoker.ThrowableWrapper) ex.getCause();
			}
		} else {
			return invokeOperation(invoker);
		}
	} 
	// ... ...
} // end execute
```
### (7). 查看Cache.get签名
```
// ***************************************************
// 注意看,这里有synchronized关键字.
// ***************************************************
public synchronized <T> T get(Object key, Callable<T> valueLoader) {
	ValueWrapper result = get(key);

	if (result != null) {
		return (T) result.get();
	}
     
	// Callable是个回调,在这里才会真正的调用:目标类上的方法.
	T value = valueFromLoader(key, valueLoader);
	put(key, value);
	return value;
} // end 
```

### (8). 总结
> Spring Cache用到了哪些设计模式?   
> 1. 组合模式(CompositeCacheManager),话说这东西没有太大的用处哈.     
> 2. 策略模式(KeyGenerator).   
> 3. 解释器模式(CacheResolver).   
> 4. 代理模式(CacheInterceptor).  
> 总体来说,Spring Cache的源码有一点点的乱,而且,可扩展性方面做得真的是一般般.  