---
layout: post
title: 'Spring Boot与Kafka集成源码之@KafkaListener(八)' 
date: 2021-10-01
author: 李新
tags:  Kafka
---

### (1). 概述
由于SpringBoot对Kafka进行了包装,只是留出了一个注解给我们了,在这一小篇,主要剖析:@KafkaListener注解,以及了解该注解能支持哪些参数签名.     

### (2). 如何找到突破口?
```
# SpringBoot启动时,会回调:EnableAutoConfiguration配置的所有类
# spring-boot-autoconfigure.jar/META-INF/spring.factories

org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.kafka.KafkaAutoConfiguration
```
### (3). 如何找到突破口?
```
@Import({ 
	// **************************************************
	// 1. Import了另外一个配置类:KafkaAnnotationDrivenConfiguration
	// **************************************************
	KafkaAnnotationDrivenConfiguration.class,
	KafkaStreamsAnnotationDrivenConfiguration.class })
public class KafkaAutoConfiguration {
	// ... ...
}
```
### (4). KafkaAnnotationDrivenConfiguration
```
@Configuration
@ConditionalOnClass(EnableKafka.class)
class KafkaAnnotationDrivenConfiguration {
	
	@Configuration
	// *******************************************************************
	// 1. 启用了:@EnableKafka注解
	// *******************************************************************
	@EnableKafka
	@ConditionalOnMissingBean(name = KafkaListenerConfigUtils.KAFKA_LISTENER_ANNOTATION_PROCESSOR_BEAN_NAME)
	protected static class EnableKafkaConfiguration {

	}
	
}	
```
### (5). @EnableKafka
```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
// *********************************************************
// 又导入了另外一个配置类:KafkaBootstrapConfiguration
// *********************************************************
@Import(KafkaBootstrapConfiguration.class)
public @interface EnableKafka {
	
}
```
### (6). KafkaBootstrapConfiguration
```
@Configuration
public class KafkaBootstrapConfiguration {

	// **************************************************************************
	// KafkaListenerAnnotationBeanPostProcessor
	// **************************************************************************
	@Bean(name = KafkaListenerConfigUtils.KAFKA_LISTENER_ANNOTATION_PROCESSOR_BEAN_NAME)
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public KafkaListenerAnnotationBeanPostProcessor kafkaListenerAnnotationProcessor() {
		return new KafkaListenerAnnotationBeanPostProcessor();
	}
```
### (7). KafkaListenerAnnotationBeanPostProcessor.postProcessAfterInitialization
> 通过剖析源码得知,postProcessAfterInitialization方法主要是把注解收集,并转化在业务模型:MethodKafkaListenerEndpoint,并把所有的结果保存在:KafkaListenerEndpointRegistrar里.    
> 那么:底层又是如何收到消息后,把消息回调给:MethodKafkaListenerEndpoint呢?

```
public class KafkaListenerAnnotationBeanPostProcessor<K, V>
        // ***********************************************************
		// BeanPostProcessor有两个回调函数,Spring每次new出对象后,都会回调给:BeanPostProcessor
		// 所以,我们的重点是关注:BeanPostProcessor.postProcessAfterInitialization,因为这里是AOP织入的核心哈!
		// ***********************************************************
		implements BeanPostProcessor, 
		           Ordered, 
				   // BeanFactoryAware的作用,要求Spring注入:BeanFactory
				   BeanFactoryAware, 
				   SmartInitializingSingleton {

	
	// *************************************************************
	// 1. Spring核心回调方法
	// *************************************************************
	@Override
	public Object postProcessAfterInitialization(final Object bean, final String beanName) throws BeansException {
		// 2. 因为,每一个Bean都会回调这个方法,所以,做了一个去重,避免重复反射获得注解信息,而又空无一获.
		if (!this.nonAnnotatedClasses.contains(bean.getClass())) {
			// 3. 获得Bean对应的Class信息
			Class<?> targetClass = AopUtils.getTargetClass(bean);
			
			// 4. 查找类级别上的注解(@KafkaListener/@KafkaListeners)
			Collection<KafkaListener> classLevelListeners = findListenerAnnotations(targetClass);
			final boolean hasClassLevelListeners = classLevelListeners.size() > 0;
			
			// 5. 查找出方法级别上的注解(@KafkaListener/@KafkaListeners)
			final List<Method> multiMethods = new ArrayList<Method>();
			Map<Method, Set<KafkaListener>> annotatedMethods = MethodIntrospector.selectMethods(targetClass,
				new MethodIntrospector.MetadataLookup<Set<KafkaListener>>() {

					@Override
					public Set<KafkaListener> inspect(Method method) {
						Set<KafkaListener> listenerMethods = findListenerAnnotations(method);
						return (!listenerMethods.isEmpty() ? listenerMethods : null);
					}

			}); // end annotatedMethods
			
			// 6. 过滤出:方法级别上,拥有注解(@KafkaHandler)的方法.
			if (hasClassLevelListeners) {
				Set<Method> methodsWithHandler = MethodIntrospector.selectMethods(targetClass,
						(ReflectionUtils.MethodFilter) method ->
								AnnotationUtils.findAnnotation(method, KafkaHandler.class) != null);
				multiMethods.addAll(methodsWithHandler);
			}
			
			// 7. 当方法上没有注解时,通过集合(annotatedMethods)记录,防止下次再对这个对象进行解析.
			if (annotatedMethods.isEmpty()) {
				this.nonAnnotatedClasses.add(bean.getClass());
				if (this.logger.isTraceEnabled()) {
					this.logger.trace("No @KafkaListener annotations found on bean type: " + bean.getClass());
				}
			}
			else {
				// *******************************************************************
				// 8. 对方法级别上的注解进行处理
				// *******************************************************************
				// Non-empty set of methods
				for (Map.Entry<Method, Set<KafkaListener>> entry : annotatedMethods.entrySet()) {
					// key = Method
					// value = @KafkaListener
					Method method = entry.getKey();
					for (KafkaListener listener : entry.getValue()) {
						// ************************************************************
						// 9. 处理:@KafkaListener
						// ************************************************************
						processKafkaListener(listener, method, bean, beanName);
					}
				}
				
				if (this.logger.isDebugEnabled()) {
					this.logger.debug(annotatedMethods.size() + " @KafkaListener methods processed on bean '"
							+ beanName + "': " + annotatedMethods);
				}
			}
			
			// 9. 对类级别上的注解进行处理.
			if (hasClassLevelListeners) {
				processMultiMethodListeners(classLevelListeners, multiMethods, bean, beanName);
			}
		}
		return bean;
	} // end postProcessAfterInitialization

    // 获取方法上的注解(@KafkaListener/@KafkaListeners)
	// 从代码上能看出来,是支持配置多个消费者的.
	private Collection<KafkaListener> findListenerAnnotations(Class<?> clazz) {
		Set<KafkaListener> listeners = new HashSet<KafkaListener>();
		KafkaListener ann = AnnotationUtils.findAnnotation(clazz, KafkaListener.class);
		if (ann != null) {
			listeners.add(ann);
		}
		KafkaListeners anns = AnnotationUtils.findAnnotation(clazz, KafkaListeners.class);
		if (anns != null) {
			listeners.addAll(Arrays.asList(anns.value()));
		}
		return listeners;
	} // end findListenerAnnotations
	
	// **************************************************************
	// 9.1 处理@KafkaListener
	// **************************************************************
	protected void processKafkaListener(KafkaListener kafkaListener, Method method, Object bean, String beanName) {
		Method methodToUse = checkProxy(method, bean);
		
		// 开始,解析注解:@KafkaListener,把解析的结果通过:MethodKafkaListenerEndpoint包裹着:Method/BeanFactory/KafkaListenerErrorHandler
		MethodKafkaListenerEndpoint<K, V> endpoint = new MethodKafkaListenerEndpoint<K, V>();
		endpoint.setMethod(methodToUse);
		endpoint.setBeanFactory(this.beanFactory);
		
		// 解析注解(@KafkaListener(errorHandler="${errorHandler}"))上的错误处理定义.
		String errorHandlerBeanName = resolveExpressionAsString(kafkaListener.errorHandler(), "errorHandler");
		if (StringUtils.hasText(errorHandlerBeanName)) {
			endpoint.setErrorHandler(this.beanFactory.getBean(errorHandlerBeanName, KafkaListenerErrorHandler.class));
		}
		
		// ***************************************************************
		// 9.2 委托给另一个方法,处理@KafkaListener
		// ***************************************************************
		processListener(endpoint, kafkaListener, bean, methodToUse, beanName);
	} // end processKafkaListener
	
	// ***************************************************************
	// 9.3 processListener
	// ***************************************************************
	protected void processListener(
	             MethodKafkaListenerEndpoint<?, ?> endpoint, 
				 KafkaListener kafkaListener, 
				 Object bean,
				Object adminTarget, 
				String beanName) {
		
		
		String beanRef = kafkaListener.beanRef();
		if (StringUtils.hasText(beanRef)) {
			this.listenerScope.addListener(beanRef, bean);
		}
		
		// ********************************************************************
		// MethodKafkaListenerEndpoint承载着解析@KafkaListener后的结果,即:KafkaListener对应的业务模型是:MethodKafkaListenerEndpoint
		// ********************************************************************
		endpoint.setBean(bean);
		endpoint.setMessageHandlerMethodFactory(this.messageHandlerMethodFactory);
		endpoint.setId(getEndpointId(kafkaListener));
		// 解析groupid,支持EL表达式
		endpoint.setGroupId(getEndpointGroupId(kafkaListener, endpoint.getId()));
		// 解析topics,仅topic支持EL表达式
		endpoint.setTopicPartitions(resolveTopicPartitions(kafkaListener));
		// 解析topic,支持EL表达式
		endpoint.setTopics(resolveTopics(kafkaListener));
		// 支持EL表达式
		endpoint.setTopicPattern(resolvePattern(kafkaListener));
		endpoint.setClientIdPrefix(resolveExpressionAsString(kafkaListener.clientIdPrefix(),
				"clientIdPrefix"));
		String group = kafkaListener.containerGroup();
		if (StringUtils.hasText(group)) {
			Object resolvedGroup = resolveExpression(group);
			if (resolvedGroup instanceof String) {
				endpoint.setGroup((String) resolvedGroup);
			}
		}
		
		// ******************************************************
		// 解析消费者组对应的:并发数,建议要与partition数量相同.
		// 这就,解释通了,为什么在注解上,concurrency是字符串类型,而不是直接数值类型,就是为了支持动态化.
		// ******************************************************
		String concurrency = kafkaListener.concurrency();
		if (StringUtils.hasText(concurrency)) {
			endpoint.setConcurrency(resolveExpressionAsInteger(concurrency, "concurrency"));
		}
		
		
		String autoStartup = kafkaListener.autoStartup();
		if (StringUtils.hasText(autoStartup)) {
			endpoint.setAutoStartup(resolveExpressionAsBoolean(autoStartup, "autoStartup"));
		}

		// 解析containerFactory(KafkaListenerContainerFactory)
		KafkaListenerContainerFactory<?> factory = null;
		String containerFactoryBeanName = resolve(kafkaListener.containerFactory());
		if (StringUtils.hasText(containerFactoryBeanName)) {
			Assert.state(this.beanFactory != null, "BeanFactory must be set to obtain container factory by bean name");
			try {
				factory = this.beanFactory.getBean(containerFactoryBeanName, KafkaListenerContainerFactory.class);
			}
			catch (NoSuchBeanDefinitionException ex) {
				throw new BeanInitializationException("Could not register Kafka listener endpoint on [" + adminTarget
						+ "] for bean " + beanName + ", no " + KafkaListenerContainerFactory.class.getSimpleName()
						+ " with id '" + containerFactoryBeanName + "' was found in the application context", ex);
			}
		}
		
		// ****************************************************************
		// 仅仅是把@KafkaListener注解的信息,转换成了业务模型对象:MethodKafkaListenerEndpoint,并保存着.
		// ****************************************************************
		this.registrar.registerEndpoint(endpoint, factory);
		if (StringUtils.hasText(beanRef)) {
			this.listenerScope.removeListener(beanRef);
		}
	} // end processListener
}
```
### (8). KafkaListenerAnnotationBeanPostProcessor.afterSingletonsInstantiated
```
// afterSingletonsInstantiated是Spring回调
public void afterSingletonsInstantiated() {
	
	this.registrar.setBeanFactory(this.beanFactory);
	
	
	// 处理Spring容器中所有的:KafkaListenerConfigurer类的实现类.
	if (this.beanFactory instanceof ListableBeanFactory) {
		Map<String, KafkaListenerConfigurer> instances =
				((ListableBeanFactory) this.beanFactory).getBeansOfType(KafkaListenerConfigurer.class);
		for (KafkaListenerConfigurer configurer : instances.values()) {
			configurer.configureKafkaListeners(this.registrar);
		}
	}

	// ... ...
	// ********************************************************************
	// 上一个方法,仅仅是把注解转化成了业务模型,然后保存在:registrar
	// 在这个方法里就能看到:会调用registrar.afterPropertiesSet进行初始化.
	// ********************************************************************
	// Actually register all listeners
	this.registrar.afterPropertiesSet();
}
```
### (9). KafkaListenerEndpointRegistrar.afterPropertiesSet
```
public class KafkaListenerEndpointRegistrar implements BeanFactoryAware, InitializingBean {
	
	public void afterPropertiesSet() {
		// ****************************************
		// 委托给了另一个方法:registerAllEndpoints
		// ****************************************
		registerAllEndpoints();
	}
		
	protected void registerAllEndpoints() {
		// endpointDescriptors包裹着:KafkaListenerEndpoint
		synchronized (this.endpointDescriptors) {
			for (KafkaListenerEndpointDescriptor descriptor : this.endpointDescriptors) {
				// ************************************************************
				// 又委托给了:endpointRegistry.registerListenerContainer方法
				// ************************************************************
				this.endpointRegistry.registerListenerContainer(descriptor.endpoint, resolveContainerFactory(descriptor));
			}
			// 标记初始化完成
			this.startImmediately = true;  // trigger immediate startup
		}
	}
	
}	
```
### (10). KafkaListenerEndpointRegistry.registerListenerContainer
```
public class KafkaListenerEndpointRegistry 
       implements DisposableBean, SmartLifecycle, ApplicationContextAware, ApplicationListener<ContextRefreshedEvent> {
    
    // 通过KafkaListenerContainerFactory创建监听处理.
	public void registerListenerContainer(KafkaListenerEndpoint endpoint, KafkaListenerContainerFactory<?> factory) {
		registerListenerContainer(endpoint, factory, false);
	} // end registerListenerContainer
	
	public void registerListenerContainer(
	          KafkaListenerEndpoint endpoint, 
			  KafkaListenerContainerFactory<?> factory,
			  // false
			  boolean startImmediately) {
		Assert.notNull(endpoint, "Endpoint must not be null");
		Assert.notNull(factory, "Factory must not be null");

		String id = endpoint.getId();
		Assert.hasText(id, "Endpoint id must not be empty");
		synchronized (this.listenerContainers) {
			Assert.state(!this.listenerContainers.containsKey(id),
					"Another endpoint is already registered with id '" + id + "'");
            // *******************************************************************
			// 这一步很重要:把业务模型(KafkaListenerEndpoint)转换成:MessageListenerContainer对象
			// MessageListenerContainer对象是:org.springframework.context.SmartLifecycle的实现类.
			// 所以,只要把MessageListenerContainer向Spring中注册,就会自动回调:
			// SmartLifecycle.start方法,所以,只需要关注:MessageListenerContainer.start方法即可
			// *******************************************************************
			MessageListenerContainer container = createListenerContainer(endpoint, factory);
			this.listenerContainers.put(id, container);
			if (StringUtils.hasText(endpoint.getGroup()) && this.applicationContext != null) {
				List<MessageListenerContainer> containerGroup;
				if (this.applicationContext.containsBean(endpoint.getGroup())) {
					containerGroup = this.applicationContext.getBean(endpoint.getGroup(), List.class);
				} else {
					containerGroup = new ArrayList<MessageListenerContainer>();
					// *********************************************************************************
					// 向Spring容器中注册一个Bean
					// *********************************************************************************
					this.applicationContext.getBeanFactory().registerSingleton(endpoint.getGroup(), containerGroup);
				}
				containerGroup.add(container);
			}
			if (startImmediately) {
				startIfNecessary(container);
			}
		}
	} // end registerListenerContainer
}			
```
### (11). 总结
```
# 1.  KafkaListenerAnnotationBeanPostProcessor.postProcessAfterInitialization方法,负责收集所有的注解(@KafkaListener),并转化成临时业务模型对象:KafkaListenerEndpoint,并把结果保存在:KafkaListenerEndpointRegistrar对象里.
# 2.  KafkaListenerAnnotationBeanPostProcessor.afterSingletonsInstantiated方法,会触发:KafkaListenerEndpointRegistrar.afterPropertiesSet对所有的业务模型进行处理.
# 3.  KafkaListenerEndpointRegistrar.afterPropertiesSet方法,遍历所有的业务模型:KafkaListenerEndpoint,委托给:KafkaListenerEndpointRegistry.registerListenerContainer方法.  
# 4.  KafkaListenerEndpointRegistry.registerListenerContainer会把:KafkaListenerEndpoint转换成:MessageListenerContainer.
# 5.  MessageListenerContainer 是一个接口,实现类为:org.springframework.kafka.listener.ConcurrentMessageListenerContainer
# 同时,MessageListenerContainer属于:org.springframework.context.SmartLifecycle的实现类,所以,把MessageListenerContainer向Spring容器
# 注册即可,Spring容器会回调MessageListenerContainer.start()方法,监听Kafka事件.  
```