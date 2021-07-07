---
layout: post
title: 'Spring Cache源码剖析(二)'
date: 2021-07-05
author: 李新
tags:  Spring 
---

### (1). 前言
在前面对Spring+Redis有了一个简单的案例入门,在这里将对Spring Cache部份的源码进行剖析.  

### (2). Spring Cache类图
> CacheManager固名思义:就是对Cache进行管理,可以管理多个Cache.         
> Cache是对缓存的一些基本操作(添加数据到缓存/剔除中的缓存/清空整个缓存...)         
> CacheResolver在AOP时会需要根据注解,解析出Cache对象.        
> KeyGenerator用于缓存时自定义KEY策略,注意,与注解上的key是互斥的.         

!["CacheManager"](/assets/spring/imgs/spring_CacheManager.jpg)    
!["Cache"](/assets/spring/imgs/spring_Cache.jpg)    
!["CacheResolver"](/assets/spring/imgs/spring_CacheResolver.jpg)   
!["KeyGenerator"](/assets/spring/imgs/spring_KeyGenerator.jpg)   

### (3). @Cacheable注解
> 使用@Cacheable标注的方法,Spring在每次执行前都会检查Cache中是否存在相同key的缓存元素,如果存在就不再执行该方法,而是直接从缓存中获取结果进行返回,否则才会执行并将返回结果存入指定的缓存中.   

```
@Cacheable(
  // 定义缓存的名称
  cacheNames="users",
  
  // 使用key属性自定义key
  // #p0                : 通过下标读取参数的第一个元数
  // #p0.id             : 通过下标读取参数的第一个元数的属性id
  // #user.id           : 通过参数名称读取属性id
  // #root.methodName   : 当前方法名
  // #root.method.name  : 当前方法
  // #root.target       : 目标对象
  // #root.targetClass  : 目标Class
  // #root.args[0]      : 方法参数组
  // #root.caches[0].name   : 当前被调用的方法使用的Cache(caceNames)
  key="#p0" , 
  
  // 通过condition指定符合条件,则缓存,不符合条件的数据,则不缓存.
  condition = "#user.id % 2 == 0" ,
  
  // 告诉底层的缓存提供者将缓存的入口锁住,这样就只能有一个线程计算操作的结果值,而其它线程需要等待,这样就避免了 n-1 次数据库访问.
  // sync=true,可以避免缓存击穿问题
  // Cache.get(Object key, Callable<T> valueLoader)
  // 实际上是依赖于java的synchronized关键字
  // synchronized <T> T get(Object key, Callable<T> valueLoader)
  // sync = false
)
```
### (4). @CachePut注解
> @CachePut与@Cacheable不同的是,@CachePut标注的方法在执行前不会去检查缓存中是否存在之前执行过的结果,而是每次都会执行该方法,并将执行结果以键值对的形式存入指定的缓存中.  

### (5). @CacheEvict注解
>  @CacheEvict是用来标注在需要清除缓存元素

```
@CacheEvict(
  // allEntries为true时,Spring Cache将忽略指定的key,清除所有的元素.
  allEntries = false,
  // 清除操作默认是在"调用代理类的方法成功后"执行之后触发清除操作.
  // 如果被代理后的方法抛出异常而未能成功返回时,则不会触发清除操作.
  // 使用beforeInvocation可以改变触发清除操作的时间.
  // 当我们指定该属性值为true时,Spring会在调用"代理方法之前"清除缓存中的指定元素. 
  beforeInvocation = false
)
```
### (6). @Caching注解
> @Caching注解可以让我们在一个方法或者类上同时指定多个Spring Cache相关的注解.
> 其拥有三个属性:cacheable、put和evict,分别用于指定@Cacheable、@CachePut和@CacheEvict. 

### (7). Spring Cache配置类在哪?
```
# **************************************ProxyCachingConfiguration**********************************************************
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

	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public CacheOperationSource cacheOperationSource() {
		return new AnnotationCacheOperationSource();
	}

	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public CacheInterceptor cacheInterceptor() {
		CacheInterceptor interceptor = new CacheInterceptor();
		interceptor.configure(this.errorHandler, this.keyGenerator, this.cacheResolver, this.cacheManager);
		interceptor.setCacheOperationSource(cacheOperationSource());
		return interceptor;
	}
}


# 默认的CacheManager是ConcurrentMapCacheManager
# **************************************SimpleCacheConfiguration**********************************************************
@Configuration
@ConditionalOnMissingBean(CacheManager.class)
@Conditional(CacheCondition.class)
class SimpleCacheConfiguration {

	private final CacheProperties cacheProperties;

	private final CacheManagerCustomizers customizerInvoker;

	SimpleCacheConfiguration(CacheProperties cacheProperties,
			CacheManagerCustomizers customizerInvoker) {
		this.cacheProperties = cacheProperties;
		this.customizerInvoker = customizerInvoker;
	}

	@Bean
	public ConcurrentMapCacheManager cacheManager() {
		ConcurrentMapCacheManager cacheManager = new ConcurrentMapCacheManager();
		List<String> cacheNames = this.cacheProperties.getCacheNames();
		if (!cacheNames.isEmpty()) {
			cacheManager.setCacheNames(cacheNames);
		}
		return this.customizerInvoker.customize(cacheManager);
	}

}
```
### (8). Spring Cache AOP逻辑代码
> Spring Cache 的原理是AOP,那么,入口类在哪呢? 
> 答案就是: CacheInterceptor

```
public class CacheInterceptor 
       extends CacheAspectSupport 
	   // ***********************************************************
	   // 1. MethodInterceptor 
	   // ***********************************************************
	   implements MethodInterceptor, Serializable {

	@Override
	@Nullable
	public Object invoke(final MethodInvocation invocation) throws Throwable {
		Method method = invocation.getMethod();

        // 2. 创建:CacheOperationInvoker对象,并没有触发里面的方法调用.
		CacheOperationInvoker aopAllianceInvoker = () -> {
			try {
				return invocation.proceed();
			}
			catch (Throwable ex) {
				throw new CacheOperationInvoker.ThrowableWrapper(ex);
			}
		};
		

		try {
			// 3. 会读取注解上的信息,然后,触发目标对象的调用,再把结果进行缓存.
			return execute(aopAllianceInvoker, invocation.getThis(), method, invocation.getArguments());
		} catch (CacheOperationInvoker.ThrowableWrapper th) {
			throw th.getOriginal();
		}
	}
}
```

### (9). Spring Cache 如何避免缓存穿透
> @Cacheable(cacheNames = "users" , sync = true),当对相同的key进行操作时,会运用Java的synchronized关键字,只同一个进程里,只会允许一个线程去访问DB.  

```
# ********************************************************CacheAspectSupport********************************************************
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
					  // 看下方法签名,get方法需要一个回调函数.
					  // cache.get(Object key, Callable<T> valueLoader)
					  // *************************************************************************
				      cache.get(key, () -> unwrapReturnValue(invokeOperation(invoker)))
				);
			}
			catch (Cache.ValueRetrievalException ex) {
				throw (CacheOperationInvoker.ThrowableWrapper) ex.getCause();
			}
		}
		else {
			// No caching required, only call the underlying method
			return invokeOperation(invoker);
		}
	} 
	// ... ...
} // end execute
		
# ********************************************************RedisCache********************************************************		
// 注意看,这里有synchronized关键字.
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
### (10). 总结
> Spring Cache用到了哪些设计模式?   
> 1. 组合模式(CompositeCacheManager).   
> 2. 策略模式(KeyGenerator).   
> 3. 解释器模式(CacheResolver).   
> 4. 代理模式(CacheInterceptor).  