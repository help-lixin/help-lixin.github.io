---
layout: post
title: 'Spring StateMachine源码StateMachineAnnotationPostProcessorConfiguration(三)' 
date: 2022-09-03
author: 李新
tags:  SpringStateMachine
---

### (1). 概述
还有一小部份内容没有剖析完毕,那就是,状态发生变化后,触发action的处理,Spring是如何识这些注解的?

### (2). @WithStateMachine
```
@WithStateMachine(name="orderSingleMachine")
public class OrderSingleEventConfig {
	
    /**
     * 当前状态UNPAID
     */
    @OnTransition(target = "UNPAID")
    public void create() {
        logger.info("---订单创建，待支付---");
    }
    // ... ... 
}
```
### (3). 入口在哪?
```
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Inherited
@Documented
@Component
@Configuration
// ***********************************************************************
// 导入一个配置到类Spring容器
// ***********************************************************************
@Import(StateMachineAnnotationPostProcessorConfiguration.class)
public @interface EnableWithStateMachine {
}
```
### (4). StateMachineAnnotationPostProcessorConfiguration
```
@Configuration
public class StateMachineAnnotationPostProcessorConfiguration {

	private final static String POST_PROCESSOR_BEAN_ID = "org.springframework.statemachine.processor.stateMachineAnnotationPostProcessor";

    // *************************************************************
	// 状态机后置处理
	// *************************************************************
	@Bean(name = POST_PROCESSOR_BEAN_ID)
	public StateMachineAnnotationPostProcessor springStateMachineAnnotationPostProcessor() {
		return new StateMachineAnnotationPostProcessor();
	}
}
```
### (5). StateMachineAnnotationPostProcessor
```
public class StateMachineAnnotationPostProcessor implements 
         // *********************************************************************
		 // 1. BeanPostProcessor很重要,Spring通过反射创建对象实例后,会回调实现BeanPostProcessor的相关方法来着的.
		 // *********************************************************************
        BeanPostProcessor, BeanFactoryAware, InitializingBean,
		Lifecycle, ApplicationListener<ApplicationEvent> {

  
     // ***********************************************************
     // 2. 为每一个注解,配置一个处理类.
	 // ***********************************************************
	@Override
	public void afterPropertiesSet() {
		Assert.notNull(beanFactory, "BeanFactory must not be null");
		postProcessors.put(OnTransition.class,
				new StateMachineActivatorAnnotationPostProcessor<OnTransition>(beanFactory));
		postProcessors.put(OnTransitionStart.class,
				new StateMachineActivatorAnnotationPostProcessor<OnTransitionStart>(beanFactory));
		postProcessors.put(OnTransitionEnd.class,
				new StateMachineActivatorAnnotationPostProcessor<OnTransitionEnd>(beanFactory));
		postProcessors.put(OnStateChanged.class,
				new StateMachineActivatorAnnotationPostProcessor<OnStateChanged>(beanFactory));
		postProcessors.put(OnStateEntry.class,
				new StateMachineActivatorAnnotationPostProcessor<OnStateEntry>(beanFactory));
		postProcessors.put(OnStateExit.class,
				new StateMachineActivatorAnnotationPostProcessor<OnStateExit>(beanFactory));
		postProcessors.put(OnStateMachineStart.class,
				new StateMachineActivatorAnnotationPostProcessor<OnStateMachineStart>(beanFactory));
		postProcessors.put(OnStateMachineStop.class,
				new StateMachineActivatorAnnotationPostProcessor<OnStateMachineStop>(beanFactory));
		postProcessors.put(OnEventNotAccepted.class,
				new StateMachineActivatorAnnotationPostProcessor<OnEventNotAccepted>(beanFactory));
		postProcessors.put(OnStateMachineError.class,
				new StateMachineActivatorAnnotationPostProcessor<OnStateMachineError>(beanFactory));
		postProcessors.put(OnExtendedStateChanged.class,
				new StateMachineActivatorAnnotationPostProcessor<OnExtendedStateChanged>(beanFactory));
	} // end afterPropertiesSet
  
    
	// ***********************************************************
	// 3. Spring创建Bean实例后,会先调用:postProcessAfterInitialization
	// ***********************************************************
	@Override
	public Object postProcessAfterInitialization(final Object bean, final String beanName) throws BeansException {
		Assert.notNull(beanFactory, "BeanFactory must not be null");
		
		// 根据实例信息(bean)获得Class信息.
		final Class<?> beanClass = getBeanClass(bean);
		// 看下类上是否有注解@WithStateMachine
		if (AnnotationUtils.findAnnotation(beanClass, WithStateMachine.class) == null) {
			// we only post-process beans having WithStateMachine
			// in it or as a meta annotation
			return bean;
		}

		// ***********************************************************
		// 4. 遍历类上所有的方法,对注解进行解析,然后,对注解上的方法通过:StateMachineHandler进行装饰.
		// ***********************************************************
		ReflectionUtils.doWithMethods(beanClass, new ReflectionUtils.MethodCallback() {
			@SuppressWarnings({ "unchecked", "rawtypes" })
			public void doWith(Method method) throws IllegalArgumentException, IllegalAccessException {
				Map<Class<? extends Annotation>, List<Annotation>> annotationChains = new HashMap<>();
				for (Class<? extends Annotation> annotationType : postProcessors.keySet()) {
					if (AnnotatedElementUtils.isAnnotated(method, annotationType.getName())) {
						List<Annotation> annotationChain = getAnnotationChain(method, annotationType);
						if (annotationChain.size() > 0) {
							annotationChains.put(annotationType, annotationChain);
						}
					}
				}

				for (Entry<Class<? extends Annotation>, List<Annotation>> entry : annotationChains.entrySet()) {
					Class<? extends Annotation> annotationType = entry.getKey();
					List<Annotation> annotations = entry.getValue();
					Annotation metaAnnotation = null;
					Annotation annotation = null;
					if (annotations.size() == 2) {
						annotation = annotations.get(0);
						metaAnnotation = annotations.get(1);
					} else if (annotations.size() == 1) {
						annotation = annotations.get(0);
						metaAnnotation = annotations.get(0);
					}

					MethodAnnotationPostProcessor postProcessor = metaAnnotation != null ? postProcessors.get(annotationType) : null;
					if (postProcessor != null) {
						// 注解上的方法,进行Wrapper(StateMachineHandler)
						// TODO: should change post processor to handle annotation list
						Object result = postProcessor.postProcess(beanClass, bean, beanName, method, metaAnnotation, annotation);
						if (result != null && result instanceof StateMachineHandler) {
							
							// 自动生成一个Bean名称
							// orderSingleEventConfig.create.OnTransition
							String endpointBeanName = generateBeanName(beanName, method, annotation.annotationType());

							if (result instanceof BeanNameAware) {
								((BeanNameAware) result).setBeanName(endpointBeanName);
							}
							
							// ******************************************************************
							// 向Spring容器中注册一个Bean,名称是自动生成的,类型为:StateMachineHandler
							// ******************************************************************
							beanFactory.registerSingleton(endpointBeanName, result);
							if (result instanceof BeanFactoryAware) {
								((BeanFactoryAware) result).setBeanFactory(beanFactory);
							}
							if (result instanceof InitializingBean) {
								try {
									((InitializingBean) result).afterPropertiesSet();
								} catch (Exception e) {
									throw new BeanInitializationException("failed to initialize annotated component", e);
								}
							}
							if (result instanceof Lifecycle) {
								lifecycles.add((Lifecycle) result);
								if (result instanceof SmartLifecycle && ((SmartLifecycle) result).isAutoStartup()) {
									((SmartLifecycle) result).start();
								}
							}
							if (result instanceof ApplicationListener) {
								listeners.add((ApplicationListener) result);
							}
						}
					}
				}
			}
		}, ReflectionUtils.USER_DECLARED_METHODS);
		return bean;
	} // end postProcessAfterInitialization
}
```
### (6). 总结
咦,好像啥也没干,解析了注解,然后,用StateMachineHandler装饰了一下,然后,向Spring容器中注册,把action与event关联的部份应该还是在ObjectStateMachineFactory里面.  