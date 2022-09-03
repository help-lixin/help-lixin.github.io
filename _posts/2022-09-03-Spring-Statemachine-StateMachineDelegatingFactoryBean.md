---
layout: post
title: 'Spring StateMachine源码StateMachineDelegatingFactoryBean(二)' 
date: 2022-09-03
author: 李新
tags:  SpringStateMachine
---

### (1). 概述
前面分析到一个核心的类,那就是:StateMachineDelegatingFactoryBean,它是状态机的代理工厂类.

### (2). StateMachineDelegatingFactoryBean
```
private static class StateMachineDelegatingFactoryBean<S, E>
		// *******************************************************************************
		// 1. BeanDelegatingFactoryBean实现了:InitializingBean和FactoryBean
		// *******************************************************************************
		extends BeanDelegatingFactoryBean<StateMachine<S, E>,StateMachineConfigBuilder<S, E>,StateMachineConfig<S, E>>
		implements SmartLifecycle, 
		           BeanNameAware, 
				   BeanClassLoaderAware {
    
	// *******************************************************************************
	// 2.Spring回调:InitializingBean.afterPropertiesSet
	// *******************************************************************************
	public void afterPropertiesSet() throws Exception {
		// clazzName = help.lixin.StateMachineConfig
		// 通过Class.forName加载类,并且,从Spring容器中,获得这个类的实例出来.
		AnnotationConfigurer<StateMachineConfig<S, E>, StateMachineConfigBuilder<S, E>> configurer = (AnnotationConfigurer<StateMachineConfig<S, E>, StateMachineConfigBuilder<S, E>>) getBeanFactory().getBean(ClassUtils.forName(clazzName, classLoader));
		
		// *******************************************************************************
		// 3. 前面不是创建了一个空的Builde么,把StateMachineConfigurerAdapter的实现类(StateMachineConfig)的配置信息,填充到这个空的Builder里.
		//    在这一步,会回调:help.lixin.StateMachineConfig重写的相关方法来着的.
		//    其实,可以更加智能一些的做法,解析XML然后,向这个模型(Build)靠拢即可. 
		// *******************************************************************************
		getBuilder().apply(configurer);

		StateMachineConfig<S, E> stateMachineConfig = getBuilder().getOrBuild();
		TransitionsData<S, E> stateMachineTransitions = stateMachineConfig.getTransitions();
		StatesData<S, E> stateMachineStates = stateMachineConfig.getStates();
		ConfigurationData<S, E> stateMachineConfigurationConfig = stateMachineConfig.getStateMachineConfigurationConfig();

		
		ObjectStateMachineFactory<S, E> stateMachineFactory = null;
		// *******************************************************************************
		// 4. 创建状态机工厂,并为状态机本置相关信息(状态/转换/工厂/Bean名称...)
		// *******************************************************************************
		if (stateMachineConfig.getModel() != null && stateMachineConfig.getModel().getFactory() != null) {
			// 查看是否有扩展工厂,如果有的话,就用:ObjectStateMachineFactory Wrapper一层工厂
			stateMachineFactory = new ObjectStateMachineFactory<S, E>(new DefaultStateMachineModel<S, E>(stateMachineConfigurationConfig, null, null),stateMachineConfig.getModel().getFactory());
		} else {
			stateMachineFactory = new ObjectStateMachineFactory<S, E>(new DefaultStateMachineModel<S, E>(stateMachineConfigurationConfig, stateMachineStates, stateMachineTransitions), null);
		}

        // 为状态机配置工厂
		stateMachineFactory.setBeanFactory(getBeanFactory());
		// 为状态机配置事件启用
		stateMachineFactory.setContextEventsEnabled(contextEvents);
		// 为状态机配置Bean名称(orderSingleMachine)
		stateMachineFactory.setBeanName(beanName);
		stateMachineFactory.setHandleAutostartup(stateMachineConfigurationConfig.isAutoStart());
		if (stateMachineMonitor != null) {
			stateMachineFactory.setStateMachineMonitor(stateMachineMonitor);
		}
		StateMachine<S, E> stateMachine = stateMachineFactory.getStateMachine();
		this.lifecycle = (SmartLifecycle) stateMachine;
		this.disposableBean = (DisposableBean) stateMachine;
		
		// ****************************************************************************
		// 5. 配置FactoryBean.getObject方法
		// ****************************************************************************
		// 手动调用:FactoryBean.setObject,实际是配置:FactoryBean.getObject
		setObject(stateMachine);
	} // end afterPropertiesSet
}
```
### (3). 总结
StateMachineDelegatingFactoryBean的目的是委托Spring容器(FactoryBean),创建一个新的状态机:ObjectStateMachineFactory,按道理来说,后面要剖析ObjectStateMachineFactory,但是,还有一小部份东西没有剖析完,下一小章节,着重剖析注解:@WithStateMachine.  