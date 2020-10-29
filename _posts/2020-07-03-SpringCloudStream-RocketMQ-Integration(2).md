---
layout: post
title: 'Spring Cloud Stream源码(如何与RocketMQ进行集成协作)'
date: 2020-07-03
author: 李新
tags: SpringCloudStream
---

### (1).Binder
```
package org.springframework.cloud.stream.binder;

public interface Binder<T, C extends ConsumerProperties, P extends ProducerProperties> {
    Binding<T> bindConsumer(String name, String group, T inboundBindTarget, C consumerProperties);
    Binding<T> bindProducer(String name, T outboundBindTarget, P producerProperties);
}
```
### (2).BindingServiceConfiguration
```
@Configuration
@EnableConfigurationProperties({ BindingServiceProperties.class, SpringIntegrationProperties.class, StreamFunctionProperties.class })
@Import({ DestinationPublishingMetricsAutoConfiguration.class, SpelExpressionConverterConfiguration.class })
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
// **************************************************************
// 1. 
// 由于在: BinderFactoryConfiguration.binderTypeRegistry()方法中
// 创建了:BinderTypeRegistry,所以该配置类才能生效
// **************************************************************
@ConditionalOnBean(value = BinderTypeRegistry.class, search = SearchStrategy.CURRENT)
public class BindingServiceConfiguration {
   
   // *********************************************************** 
   // 2.创建bindService
   // *********************************************************** 
   @Bean
	@ConditionalOnMissingBean(search = SearchStrategy.CURRENT)
	public BindingService bindingService(BindingServiceProperties bindingServiceProperties,BinderFactory binderFactory, TaskScheduler taskScheduler) {
		return new BindingService(bindingServiceProperties, binderFactory, taskScheduler);
	}
 
   // *********************************************************** 
   // 3. 输出绑定的生命周期
   // *********************************************************** 
   @Bean
	@DependsOn("bindingService")
	public OutputBindingLifecycle outputBindingLifecycle(BindingService bindingService,
			Map<String, Bindable> bindables) {
            // {
            //   &org.springframework.cloud.stream.messaging.Source=org.springframework.cloud.stream.binding.BindableProxyFactory
            //   dynamicDestinationsBindable=org.springframework.cloud.stream.binding.DynamicDestinationsBindable
            // }
            
            // key:为我们定义的每一个接口,接口可以是多个来着的.
            // 重点在此处
		return new OutputBindingLifecycle(bindingService, bindables);
	}

   // *********************************************************** 
   // 3. 输入绑定的生命周期
   // *********************************************************** 
	@Bean
	@DependsOn("bindingService")
	public InputBindingLifecycle inputBindingLifecycle(BindingService bindingService, Map<String, Bindable> bindables) {
		return new InputBindingLifecycle(bindingService, bindables);
	}
}
```
### (3).OutputBindingLifecycle
```
public class OutputBindingLifecycle extends AbstractBindingLifecycle {
    private Collection<Binding<Object>> outputBindings = new ArrayList<Binding<Object>>();

	public OutputBindingLifecycle(BindingService bindingService, Map<String, Bindable> bindables) {
		super(bindingService, bindables);
	}
 
   @Override
	public int getPhase() {
		return Integer.MIN_VALUE + 1000;
	}

   // ************************************************************
   // 4. 由父类触发运行
   // public void start() {
	//	if (!running) {
	//		bindables.values().forEach(this::doStartWithBindable);
	//		this.running = true;
	//	}
	// }
   // ************************************************************
	@Override
	void doStartWithBindable(Bindable bindable) {
           // bindable = BindableProxyFactory
		Collection<Binding<Object>> bindableBindings = bindable.createAndBindOutputs(bindingService);
		if (!CollectionUtils.isEmpty(bindableBindings)) { // true
                // hodler住所有的binder
                this.outputBindings.addAll(bindableBindings);
		}
	}

	@Override
	void doStopWithBindable(Bindable bindable) {
		bindable.unbindOutputs(bindingService);
	}
}
```
### (4).BindableProxyFactory
```
public class BindableProxyFactory implements MethodInterceptor, FactoryBean<Object>, Bindable, InitializingBean {
    
    public Collection<Binding<Object>> createAndBindOutputs(BindingService bindingService) {
		List<Binding<Object>> bindings = new ArrayList<>();
		if (log.isDebugEnabled()) {
			log.debug(String.format("Binding outputs for %s:%s", this.namespace, this.type));
		}
  
		for (Map.Entry<String, BoundTargetHolder> boundTargetHolderEntry : this.outputHolders.entrySet()) {
			BoundTargetHolder boundTargetHolder = boundTargetHolderEntry.getValue();
			String outputTargetName = boundTargetHolderEntry.getKey();
			if (boundTargetHolderEntry.getValue().isBindable()) { // true
				if (log.isDebugEnabled()) {
					log.debug(String.format("Binding %s:%s:%s", this.namespace, this.type, outputTargetName));
				} 
                            // *********************************************************
                            // 5.委托给:BindingService
                            // *********************************************************
				bindings.add(bindingService.bindProducer(boundTargetHolder.getBoundTarget(), outputTargetName));
			}
		}
		return bindings;
	}// end createAndBindOutputs
}
```
### (5).BindingService
```
public class BindingService {

    //6.绑定提供者
    public <T> Binding<T> bindProducer(T output, String outputName) {
            // test-topic
            String bindingTarget = this.bindingServiceProperties.getBindingDestination(outputName);
            // ********************************************************************
            // 6.1 获得Binder对象
            // com.alibaba.cloud.stream.binder.rocketmq.RocketMQMessageChannelBinder
            // ********************************************************************
		Binder<T, ?, ProducerProperties> binder = (Binder<T, ?, ProducerProperties>) getBinder(outputName, output.getClass());
           // 获得:spring.cloud.stream.bindings.output的所有属性
		ProducerProperties producerProperties = this.bindingServiceProperties.getProducerProperties(outputName);
           // 处理扩展属性信息
		if (binder instanceof ExtendedPropertiesBinder) { // true
                Object extension = ((ExtendedPropertiesBinder) binder).getExtendedProducerProperties(outputName);
                ExtendedProducerProperties extendedProducerProperties = new ExtendedProducerProperties<>(extension);
                BeanUtils.copyProperties(producerProperties, extendedProducerProperties);
                producerProperties = extendedProducerProperties;
		}
		validate(producerProperties);
           // 7. 绑定
           // output = DirectWithAttributeChannel
           // bindingTarget = test-topic
           // binder = com.alibaba.cloud.stream.binder.rocketmq.RocketMQMessageChannelBinder
           // producerProperties = org.springframework.cloud.stream.binder.ExtendedProducerProperties
		Binding<T> binding = doBindProducer(output, bindingTarget, binder, producerProperties);
           // 放缓存里面
		this.producerBindings.put(outputName, binding);
		return binding;
	}// end bindProducer
 
    
   protected <T> Binder<T, ?, ?> getBinder(String channelName, Class<T> bindableType) {
        // null 
        String binderConfigurationName = this.bindingServiceProperties.getBinder(channelName);
        // *****************************************************************
        // 6.2 委托给:DefaultBinderFactory获得:Binder
        // com.alibaba.cloud.stream.binder.rocketmq.RocketMQMessageChannelBinder
        // *****************************************************************
        return binderFactory.getBinder(binderConfigurationName, bindableType);
	}// end getBinder
 
 
   // 7. 调和Binder的实现类,进行真正的绑定(改变Channel,使期能回调给RocketMQ)
    public <T> Binding<T> doBindProducer(T output, String bindingTarget, Binder<T, ?, ProducerProperties> binder,
			ProducerProperties producerProperties) {
		if (this.taskScheduler == null || this.bindingServiceProperties.getBindingRetryInterval() <= 0) { // false
			return binder.bindProducer(bindingTarget, output, producerProperties);
		} else {
			try {
                           // ******************************************************************************
                           // 7.重点.
                           // 这个时候才是真正的调用了:RocketMQMessageChannelBinder进行了绑定.
                           // 从方法入参就能猜到:会对DirectWithAttributesChannel进行改变.然后回调给RocketMQ
                           // ******************************************************************************
                           // bindingTarget = test-topic
                           // binder = com.alibaba.cloud.stream.binder.rocketmq.RocketMQMessageChannelBinder
                           // output = DirectWithAttributesChannel
                           // producerProperties = org.springframework.cloud.stream.binder.ExtendedProducerProperties
				return binder.bindProducer(bindingTarget, output, producerProperties);
                            // ******************************************************************************
                            // ******************************************************************************                            
			} catch (RuntimeException e) {
				LateBinding<T> late = new LateBinding<T>();
				rescheduleProducerBinding(output, bindingTarget, binder, producerProperties, late, e);
				return late;
			}
		}
	}// end doBindProducer 

}
```
### (6).DefaultBinderFactory
```
public class DefaultBinderFactory implements BinderFactory, DisposableBean, ApplicationContextAware {
   
    // 6.3 获取所有的Binder
    public synchronized <T> Binder<T, ?, ?> getBinder(
                    // name = null
                    String name, 
                    // org.springframework.cloud.stream.binder.Binder
                    Class<? extends T> bindingTargetType) {
             // null
		String binderName = StringUtils.hasText(name) ? name : this.defaultBinder;
           // 当前上下文不存在Binder
		Map<String, Binder> binders = this.context == null ? Collections.emptyMap() : this.context.getBeansOfType(Binder.class);
		Binder<T, ConsumerProperties, ProducerProperties> binder;

		if (StringUtils.hasText(binderName) && binders.containsKey(binderName)) { // false
                binder = (Binder<T, ConsumerProperties, ProducerProperties>) this.context.getBean(binderName);
		} else if (binders.size() == 1) { // false
                binder = binders.values().iterator().next();
		} else  if (binders.size() > 1) { // false
                // 有点奇怪,为什么不允许有多个?
                throw new IllegalStateException("Multiple binders are available, however neither default nor " + "per-destination binder name is provided. Available binders are " + binders.keySet());
		} else {
                   // ******************************************************
                   // 6.4 获取Bean
                   // com.alibaba.cloud.stream.binder.rocketmq.RocketMQMessageChannelBinder
                   // ******************************************************
			binder = this.doGetBinder(binderName, bindingTargetType);
		}
		return binder;
	}// end getBinder
 
    
    // 6.4 获取Bean
    private <T> Binder<T, ConsumerProperties, ProducerProperties> doGetBinder(
                    // null 
                    String name, 
                    // org.springframework.cloud.stream.binder.Binder
                    Class<? extends T> bindingTargetType) {
		String configurationName;
  
		if (StringUtils.isEmpty(name)) { // true
			Assert.notEmpty(this.binderConfigurations, "A default binder has been requested, but there is no binder available");
			if (!StringUtils.hasText(this.defaultBinder)) { // true
				Set<String> defaultCandidateConfigurations = new HashSet<>();
				for (Map.Entry<String, BinderConfiguration> binderConfigurationEntry : this.binderConfigurations.entrySet()) {
					if (binderConfigurationEntry.getValue().isDefaultCandidate()) {
						defaultCandidateConfigurations.add(binderConfigurationEntry.getKey());
					}
				}
    
				if (defaultCandidateConfigurations.size() == 1) { // true
                                   // rocketmq
					configurationName = defaultCandidateConfigurations.iterator().next();
                                   // org.springframework.cloud.stream.messaging.DirectWithAttributesChannel
                                   // rocketmq
					this.defaultBinderForBindingTargetType.put(bindingTargetType.getName(), configurationName);
				} else {
					List<String> candidatesForBindableType = new ArrayList<>();
					for (String defaultCandidateConfiguration : defaultCandidateConfigurations) {
						Binder<Object, ?, ?> binderInstance = getBinderInstance(defaultCandidateConfiguration);
						Class<?> binderType = GenericsUtils.getParameterType(binderInstance.getClass(), Binder.class, 0);
						if (binderType.isAssignableFrom(bindingTargetType)) {
							candidatesForBindableType.add(defaultCandidateConfiguration);
						}
					}
					if (candidatesForBindableType.size() == 1) {
						configurationName = candidatesForBindableType.iterator().next();
						this.defaultBinderForBindingTargetType.put(bindingTargetType.getName(), configurationName);
					}
					else {
						String countMsg = (candidatesForBindableType.size() == 0)
								? "are no binders"
								: "is more than one binder";
						throw new IllegalStateException(
								"A default binder has been requested, but there " + countMsg
								+ " available for '" + bindingTargetType.getName() + "' : "
								+ StringUtils.collectionToCommaDelimitedString(candidatesForBindableType)
								+ ", and no default binder has been set.");
					}
				}
			} else {
				configurationName = this.defaultBinder;
			}
		} else {
			configurationName = name;
		}
  
           // **************************************************
           // 6.5 委托给:getBinderInstance,获取Binder
           // **************************************************
		Binder<T, ConsumerProperties, ProducerProperties> binderInstance = getBinderInstance(configurationName);
		Assert.state(verifyBinderTypeMatchesTarget(binderInstance, bindingTargetType),
				"The binder '" + configurationName + "' cannot bind a " + bindingTargetType.getName());
		return binderInstance;
	}// end doGetBinder

    // **************************************************
    // 6.5 getBinderInstance方法.获取Binder
    // **************************************************
    // configurationName = rocketmq
    private <T> Binder<T, ConsumerProperties, ProducerProperties> getBinderInstance(String configurationName) {
          // 从缓存中获取
		if (!this.binderInstanceCache.containsKey(configurationName)) { // false
                  // org.springframework.cloud.stream.binder.BinderConfiguration
			BinderConfiguration binderConfiguration = this.binderConfigurations.get(configurationName);
			Assert.state(binderConfiguration != null, "Unknown binder configuration: " + configurationName);
                   // binderConfiguration.getBinderType() = rocketmq
                   // binderType.defaultName = rocketmq
                   // binderType.configurationClasses = [class com.alibaba.cloud.stream.binder.rocketmq.config.RocketMQBinderAutoConfiguration]
			BinderType binderType = this.binderTypeRegistry.get(binderConfiguration.getBinderType());
			Assert.notNull(binderType, "Binder type " + binderConfiguration.getBinderType() + " is not defined");
                   
                   // 填充:binderConfiguration.getProperties() 到binderProperties
			Map<String, String> binderProperties = new HashMap<>();
			this.flatten(null, binderConfiguration.getProperties(), binderProperties);
                   // 
			ArrayList<String> args = new ArrayList<>();
			for (Map.Entry<String, String> property : binderProperties.entrySet()) {
				args.add(String.format("--%s=%s", property.getKey(), property.getValue()));
			}
   
                   // 获取环境变量
			ConfigurableEnvironment environment = this.context != null ? this.context.getEnvironment() : null;
                   // null
			String defaultDomain = environment != null ? environment.getProperty("spring.jmx.default-domain") : "";
                   // --spring.jmx.default-domain=nullbinder.rocketmq
			args.add("--spring.jmx.default-domain=" + defaultDomain + "binder." + configurationName);
   
                   // *******************************************************
                   // 6.6 构建: SpringApplicationBuilder
                   // *******************************************************
			SpringApplicationBuilder springApplicationBuilder = new SpringApplicationBuilder(SpelExpressionConverterConfiguration.class)
                                   // [class com.alibaba.cloud.stream.binder.rocketmq.config.RocketMQBinderAutoConfiguration]
					.sources(binderType.getConfigurationClasses())
                                   // 禁用banner
					.bannerMode(Mode.OFF)
                                   // 日志关闭
					.logStartupInfo(false)
                                   // 非web
					.web(WebApplicationType.NONE);
                  // true
			boolean useApplicationContextAsParent = binderProperties.isEmpty() && this.context != null;
			if (useApplicationContextAsParent) { // true
				springApplicationBuilder.parent(this.context);
			}
                   
			if (environment != null && !useApplicationContextAsParent){ // false
				springApplicationBuilder.initializers(new InitializerWithOuterContext(this.context));
			}

			if (environment != null && (useApplicationContextAsParent || binderConfiguration.isInheritEnvironment())) { // true
                          // 构建StandardEnvironment
				StandardEnvironment  binderEnvironment = new StandardEnvironment();
                          // 将parent env与StandardEnvironment 进行合并
				binderEnvironment.merge(environment);
                          // 设置环境变量
				springApplicationBuilder.environment(binderEnvironment);
			}
   
                  // ******************************************************
                  // 构建出一个新的ApplicationContext,加载:com.alibaba.cloud.stream.binder.rocketmq.config.RocketMQBinderAutoConfiguration
                  // ApplicationContext ctx = new AnnotationConfigApplicationContext(com.alibaba.cloud.stream.binder.rocketmq.config.RocketMQBinderAutoConfiguration.class);
                  // Binder binder = ctx.getBean(Binder.class);
                  // ******************************************************
			ConfigurableApplicationContext binderProducingContext = springApplicationBuilder.run(args.toArray(new String[args.size()]));
                  //   从新的上下文中获得Binder
                  // com.alibaba.cloud.stream.binder.rocketmq.RocketMQMessageChannelBinder
			Binder<T, ?, ?> binder = binderProducingContext.getBean(Binder.class);
			if (this.context != null && binder instanceof ApplicationContextAware) { // true
                           // 为Binder设置context
				((ApplicationContextAware)binder).setApplicationContext(this.context);
			}
                   
                   // [org.springframework.cloud.stream.config.BindersHealthIndicatorAutoConfiguration$BindersHealthIndicatorListener]
			if (!CollectionUtils.isEmpty(this.listeners)) { // true
				for (Listener binderFactoryListener : listeners) {
                                  // rocketmq
                                  // org.springframework.context.annotation.AnnotationConfigApplicationContext
					binderFactoryListener.afterBinderContextInitialized(configurationName, binderProducingContext);
				}
			}
                   // 添加到缓存中
                   // configurationName = rocketmq
                   // SimpleImmutableEntry.key = com.alibaba.cloud.stream.binder.rocketmq.RocketMQMessageChannelBinder
                   // SimpleImmutableEntry.value = org.springframework.context.annotation.AnnotationConfigApplicationContext
			this.binderInstanceCache.put(configurationName, new SimpleImmutableEntry<>(binder, binderProducingContext));
		}
  
           // 从缓存中获取Binder
		return (Binder<T, ConsumerProperties, ProducerProperties>) this.binderInstanceCache.get(configurationName).getKey();
	}// end getBinderInstance
}
```
### (7).总结

>  1. 解析spring.binders,生成:DefaultBinderTypeRegistry
BinderFactoryConfiguration的binderTypeRegistry方法,会解析:META-INF/spring.binders文件,并产生一个bean:DefaultBinderTypeRegistry
DefaultBinderTypeRegistry内部结构为:
defaultName:rocketmq
configurationClasses:com.alibaba.cloud.stream.binder.rocketmq.config.RocketMQBinderAutoConfiguration
> 2. @EnableBinding({ Source.class }) 为定义的接口创建代理工厂
每一个声明的(Source)接口,都会对应一个BindableProxyFactory,并由它去创建相应的代理类,而它实现了FactoryBean/InitializingBean/Bindable.
afterPropertiesSet方法会解析接口上方法的注解@Output(value="output"),并创建:DirectWithAttributesChannel.同时在BindableProxyFactory内部holder(Map)住output与DirectWithAttributesChannel.
> 3. 创建OutputBindingLifecycle
OutputBindingLifecycle持有一个:Bindable集合(Map).而所有的BindableProxyFactory都实现了Bindable方法,所以,会找到所有的:BindableProxyFactory
同时它实现了:SmartLifecycle所以,会遍历map并调用:doStartWithBindable方法.访方法会委托给:BindableProxyFactory.createAndBindOutputs进行处理.
> 4. BindableProxyFactory
BindableProxyFactory会遍历第二步中holder(Map),并委托给:BindingService进行绑定提供者.
> 5. BindingService    
>     BindingService会委托给:DefaultBinderFactory获得Binder    
>     DefaultBinderFactory内部会创建ApplicationContext并加载,第一步中收集的DefaultBinderTypeRegistry(RocketMQBinderAutoConfiguration).    
>     从ApplicationContext获得Binder(RocketMQMessageChannelBinder)    
>     调用RocketMQMessageChannelBinder.bindProducer(...)进行绑定.    
>     猜测:bindProducer()方法会传递DirectWithAttributesChannel,所以在方法内部应该会对DirectWithAttributesChannel进行扩充.   
 