---
layout: post
title: 'Spring Plugin源码之PluginRegistryFactoryBean(二)' 
date: 2022-09-02
author: 李新
tags:  SpringPlugin
---

### (1). 概述
了解过Spring的都知道,在Spring中看到FactoryBean代表着这是一个工厂,必须实现3(getObject/getObjectType/isSingleton)个方法.

### (2). PluginRegistryFactoryBean
```
public class PluginRegistryFactoryBean<T extends Plugin<S>, S> 
        // ****************************************************************
		// 有模板类
		// ****************************************************************
        extends AbstractTypeAwareSupport<T>
		implements FactoryBean<PluginRegistry<T, S>> {

	
	@NonNull
	public OrderAwarePluginRegistry<T, S> getObject() {
		// 包裹Plugin的实现集合.
		return OrderAwarePluginRegistry.of(getBeans());
	}

	
	@NonNull
	public Class<?> getObjectType() {
		return OrderAwarePluginRegistry.class;
	}

	
	public boolean isSingleton() {
		return true;
	}
}
```
### (3). AbstractTypeAwareSupport
```
public abstract class AbstractTypeAwareSupport<T>
         // ***********************************************************
		 // 2. 注入了ApplicationContext
		 // ***********************************************************
		implements ApplicationContextAware, 
		           ApplicationListener<ContextRefreshedEvent>, 
				   // ***********************************************************
				   // 1. 实现了InitializingBean接口,自然,要求实现:afterPropertiesSet
				   // ***********************************************************
				   InitializingBean {

	private @Nullable ApplicationContext context;
	private @Nullable Class<T> type;
	private @Nullable BeansOfTypeTargetSource targetSource;
	private Collection<Class<?>> exclusions = Collections.emptySet();


	public void setApplicationContext(ApplicationContext context) {
		this.context = context;
	}

	
	public void setType(Class<T> type) {
		this.type = type;
	}

	
	public void setExclusions(Class<?>[] exclusions) {
		this.exclusions = Arrays.asList(exclusions);
	}

	
	@SuppressWarnings("unchecked")
	protected List<T> getBeans() {
		TargetSource targetSource = this.targetSource;
		if (targetSource == null) {
			throw new IllegalStateException("Traget source not initialized!");
		}
		// 通过ProxyFactory创建代理,要代理的对象是List,代理的实现为:BeansOfTypeTargetSource
		ProxyFactory factory = new ProxyFactory(List.class, targetSource);
		return (List<T>) factory.getProxy();
	}

    
	public void afterPropertiesSet() {
		ApplicationContext context = this.context;

		if (context == null) {
			throw new IllegalStateException("ApplicationContext not set!");
		}

		Class<?> type = this.type;

		if (type == null) {
			throw new IllegalStateException("No type configured!");
		}
		// 通过BeansOfTypeTargetSource包裹着:
		// type = Plugin
		this.targetSource = new BeansOfTypeTargetSource(context, type, false, exclusions);
	}


	static class BeansOfTypeTargetSource implements TargetSource {

		private final ListableBeanFactory context;
		private final Class<?> type;
		private final boolean eagerInit;
		private final Collection<Class<?>> exclusions;

		private boolean frozen = false;
		private @Nullable Collection<Object> components;

		public BeansOfTypeTargetSource(ListableBeanFactory context, Class<?> type, boolean eagerInit,
									   Collection<Class<?>> exclusions) {

			Assert.notNull(context, "ListableBeanFactory must not be null!");
			Assert.notNull(type, "Type must not be null!");
			Assert.notNull(exclusions, "Exclusions must not be null!");

			this.context = context;
			this.type = type;
			this.eagerInit = eagerInit;
			this.exclusions = exclusions;
			this.components = null;
		}

		
		public void freeze() {
			this.frozen = true;
		}

		
		@NonNull
		public Class<?> getTargetClass() {
			return List.class;
		}

		
		public boolean isStatic() {
			return frozen;
		}

		
		@NonNull
		@SuppressWarnings({ "rawtypes", "unchecked" })
		public synchronized Object getTarget() throws Exception {
			// ******************************************************
			// 从Spring容器中获得所有Type(Plugin)的实现类,并用ArrayList装载.
			// ******************************************************
			Collection<Object> components = this.components == null //
					? getBeansOfTypeExcept(type, exclusions) //
					: this.components;
			if (frozen && this.components == null) {
				this.components = components;
			}
			return new ArrayList(components);
		}

		
		public void releaseTarget(Object target) throws Exception {}

        
		private Collection<Object> getBeansOfTypeExcept(Class<?> type, Collection<Class<?>> exceptions) {
			// *******************************************************************
			// 从Spring容器中过滤Plugin.
			// *******************************************************************
			// 从Spring容器中,根据type找出所有的实现类,并根据type过滤.
			return Arrays.stream(context.getBeanNamesForType(type, false, eagerInit)) //
					.filter(it -> !exceptions.contains(context.getType(it))) //
					.map(it -> context.getBean(it)) //
					.collect(Collectors.toList());
		}
	}
}
```
### (4). 总结
PluginRegistryFactoryBean的目的是,找到type(Plugin)的所有实现类集合,并创建OrderAwarePluginRegistry持有(Hold)住这些Plugin的实现类集合,所以,这也就能解释通了
  为什么,要使用:PluginRegistry了,因为:OrderAwarePluginRegistry实现了PluginRegistry.  