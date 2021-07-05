---
layout: post
title: 'Spring Cache源码详解'
date: 2021-07-05
author: 李新
tags:  Spring 
---

### (1). Spring Cache是什么
Spring比较喜欢做的一件事情就是:定义规范(抽象),然后,相同类型的产品对规范进行实现(类似于:桥梁模式),可以理解:Spring Cache是Spring针对Cache定义的一套规范.    
比如:你在生活中要用到:Redis/Memcache/Tair/Guava...等等,使用Spring Cache你可以无缝自由切换(组合)这些缓存的实现.     
<font color='red'>底层实则是:对标有注解的类进行AOP拦截(注意:private/final/this call),Spring是无法代理的.</font>   

### (2). Spring Cache简单案例
```
# **************************************App**********************************************************
package help.lixin;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cache.annotation.EnableCaching;

@SpringBootApplication
// 1. 启用缓存,相当于向Spring容器中,注册一些Bean(AOP)
@EnableCaching
public class App {
	public static void main(String[] args) throws Exception {
		SpringApplication.run(App.class, args);
	}
}

# **************************************HelloService**********************************************************
package help.lixin.service.impl;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

import help.lixin.entity.User;
import help.lixin.service.IHelloService;

@Service
public class HelloService implements IHelloService {
	private Logger logger = LoggerFactory.getLogger(HelloService.class);

	private final Map<Long, List<User>> datas = new ConcurrentHashMap<Long, List<User>>();

	public HelloService() {
		init();
	}

	private void init() {
		Long tenantId = 1L;
		List<User> users = new ArrayList<User>();
		users.add(new User(1L, "张三"));
		users.add(new User(2L, "李四"));
		users.add(new User(3L, "王五"));
		users.add(new User(4L, "赵六"));
		datas.put(tenantId, users);
	}

	// 查询全部都入缓存,key=tenantId
	@Cacheable(cacheNames = "users")
	public List<User> findByTenantId(Long tenantId) {
		logger.info("query....");
		return datas.get(tenantId);
	}

	// 从我的DEBUG来看,@CacheEvict会及时更新:users缓存中key=2的缓存结果集,而不是直接把key对应的结果集给删了.
	// 当添加操作时,仅更新:tenantId % 2 == 0的缓存信息
	@CacheEvict(cacheNames = "users", condition = "#tenantId % 2 == 0" , beforeInvocation = true)
	public Boolean add(Long tenantId, User user) {
		List<User> users = new ArrayList<User>();
		users.add(user);
		if (!datas.containsKey(tenantId)) {
			datas.put(tenantId, users);
		} else {
			datas.get(tenantId).addAll(users);
		}
		return true;
	}
}

# **************************************HelloController**********************************************************
package help.lixin.controller;

import java.util.List;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

import help.lixin.entity.User;
import help.lixin.service.IHelloService;

@RestController
public class HelloController {
	private Logger logger = LoggerFactory.getLogger(HelloController.class);

	@Autowired
	private IHelloService helloService;

	@GetMapping("/{tenantId}")
	public List<User> list(@PathVariable("tenantId") Long tenantId) {
		List<User> users = helloService.findByTenantId(tenantId);
		return users;
	}

	@PostMapping("/{tenantId}")
	public String addUser(@PathVariable("tenantId") Long tenantId, @RequestBody User user) {
		helloService.add(tenantId, user);
		return "SUCCESS";
	}
}

# **************************************User**********************************************************
package help.lixin.entity;

import java.io.Serializable;

public class User implements Serializable {
	private static final long serialVersionUID = 2521452689834263244L;

	private Long id;
	private String name;
	
	public User() {
	}

	public User(Long id, String name) {
		super();
		this.id = id;
		this.name = name;
	}

	public Long getId() {
		return id;
	}

	public void setId(Long id) {
		this.id = id;
	}

	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}

	@Override
	public int hashCode() {
		final int prime = 31;
		int result = 1;
		result = prime * result + ((id == null) ? 0 : id.hashCode());
		return result;
	}

	@Override
	public boolean equals(Object obj) {
		if (this == obj)
			return true;
		if (obj == null)
			return false;
		if (getClass() != obj.getClass())
			return false;
		User other = (User) obj;
		if (id == null) {
			if (other.id != null)
				return false;
		} else if (!id.equals(other.id))
			return false;
		return true;
	}

	@Override
	public String toString() {
		return "User [id=" + id + ", name=" + name + "]";
	}

}
```
### (3). Spring Cache类图
> CacheManager固名思义,就是对Cache进行管理,可以管理多个Cache.     
> Cache是对缓存的一些基本操作(添加到缓存/缓存剔除/清空整个缓存...)      
> CacheResolver在AOP时会需要根据注解,解析出Cache对象.     
> KeyGenerator用于缓存时自定义KEY策略,注意,与注解上的key是互斥的.      

!["CacheManager"](/assets/spring/imgs/spring_CacheManager.jpg)    
!["Cache"](/assets/spring/imgs/spring_Cache.jpg)    
!["CacheResolver"](/assets/spring/imgs/spring_CacheResolver.jpg)   
!["KeyGenerator"](/assets/spring/imgs/spring_KeyGenerator.jpg)   

### (4). @Cacheable注解
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
  condition = "#user.id % 2 == 0"
)
```
### (5). @CachePut注解
> @CachePut与@Cacheable不同的是,@CachePut标注的方法在执行前不会去检查缓存中是否存在之前执行过的结果,而是每次都会执行该方法,并将执行结果以键值对的形式存入指定的缓存中.  

### (6). @CacheEvict注解
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
### (7). @Caching注解
> @Caching注解可以让我们在一个方法或者类上同时指定多个Spring Cache相关的注解.
> 其拥有三个属性:cacheable、put和evict,分别用于指定@Cacheable、@CachePut和@CacheEvict. 

### (8). Spring Cache配置类在哪?
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
```
### (9). Spring Cache AOP逻辑代码
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
### (10). 总结
> Spring Cache用到了哪些设计模式?   
> 1. 组合模式(CompositeCacheManager).   
> 2. 策略模式(KeyGenerator).   
> 3. 解释器模式(CacheResolver).   
> 4. 代理模式(CacheInterceptor).  