---
layout: post
title: 'Apollo是如何热更新配置呢(四)'
date: 2021-07-16
author: 李新
tags:  Apollo
---

### (1). 概述
在前面把Apollo与Spring进行了集成,我特意在test方法里,把this(HelloController)对象打印出来.   
目的就是想知道,当配置热更新之后,Bean实例是否会发生变化.    
<font color='red'>
结果,出乎我的意料,Bean实例没有变化,但是,Bean上的配置却更新了,瞬间就开始怀疑自己前面对@RefreshScope的剖析了.
但是,还是觉得自己应该是对的,所以,本来看源码的第一件事情就是看业务模型的,不得不先解开这个疑惑.  
答案是:Apollo没有调用Spring提供的@RefreshScope方式,对Bean进行热更新,而是,通过反射去更新Bean的.
</font> 
### (2). AutoUpdateConfigChangeListener
```
# 在日志上,会提示:AutoUpdateConfigChangeListener刷新配置成功,所以,你懂的.

public class AutoUpdateConfigChangeListener 
		// 1. 这个是Apollo定义的接口,当配置有变化时,会回调:ConfigChangeListener.onChange方法
       implements ConfigChangeListener{
	
	public void onChange(ConfigChangeEvent changeEvent) {
		
	    Set<String> keys = changeEvent.changedKeys();
		// 事件没有内容的情况下,直接返回.
	    if (CollectionUtils.isEmpty(keys)) {
	      return;
	    }
		
		// 2.遍历有更新的key(比如:jdbc.url)
	    for (String key : keys) {
		  // 3. 委托给:SpringValueRegistry.get方法,根据key,找到:SpringValue
		  //    因为,jdbc.url这个key,可能会存在于多个类里,所以,返回的是集合.
	      Collection<SpringValue> targetValues = springValueRegistry.get(beanFactory, key);
	      if (targetValues == null || targetValues.isEmpty()) {
	        continue;
	      }
	
	      for (SpringValue val : targetValues) {
			// 4. 更新内容  
	        updateSpringValue(val);
	      }
	    }
	} // end onChange

	private void updateSpringValue(SpringValue springValue) {
		try {
		   // 5.获得@Value("${jdbc.url}")注解上的内容.
		   //   调用:ConfigurableBeanFactory.resolveEmbeddedValue("${jdbc.url}")
		   //   调用这个方法的前提是,ConfigurableBeanFactory里已经是最新的配置了,估计:Apollo是先更新了配置,后再触发:ConfigChangeListener
		  Object value = resolvePropertyValue(springValue);
		  
		  // 6. 委托给:SpringValue.update方法
		  springValue.update(value);
		  logger.info("Auto update apollo changed value successfully, new value: {}, {}", value,
			  springValue);
		} catch (Throwable ex) {
		  logger.error("Auto update apollo changed value failed, {}", springValue.toString(), ex);
		}
    } // end updateSpringValue
}
```
### (3). SpringValue
```
//通过反射进行处理.
public void update(Object newVal) throws IllegalAccessException, InvocationTargetException {
	if (isField()) { 
      injectField(newVal);
    } else {
      injectMethod(newVal);
    }
} // end update

private void injectField(Object newVal) throws IllegalAccessException {
    // 获得Bean对象
	Object bean = beanRef.get();
    if (bean == null) {
      return;
    }
	// 设置允许访问private Field
    boolean accessible = field.isAccessible();
    field.setAccessible(true);
	// 通过反射设置内容
    field.set(bean, newVal);
    field.setAccessible(accessible);
}// end injectField
```
### (4). 总结
Apollo有着自己的一套热更新操作,我估摸着,它的想法是:尽可能的少依赖别人.   