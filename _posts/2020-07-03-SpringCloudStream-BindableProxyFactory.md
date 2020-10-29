---
layout: post
title: 'Spring Cloud Stream源码(BindableProxyFactory)'
date: 2020-07-03
author: 李新
tags: SpringCloudStream
---

### (1).叙述
> 在前面对@EnabledBinding的配置进行了详解,最终,会向Spring容器中注册bean.
> beanName:org.springframework.cloud.stream.messaging.Source
> value:org.springframework.cloud.stream.binding.BindableProxyFactory

### (2).BindableProxyFactory 详解
```
package org.springframework.cloud.stream.binding;
 
 // 大体的理解为.Spring会根据:type产生动态字节码.并向Spring容器中注册.
 
 public class BindableProxyFactory 
        implements
        // ********************************************** 
        // 方法拦截器
        // ********************************************** 
        MethodInterceptor, 
        // ********************************************** 
        // Spring回调,创建Bean的工厂
        // ********************************************** 
        FactoryBean<Object>, 
       // 业务接口
       Bindable, 
       // ********************************************** 
       // Spring回调,Bean实例化之后
       // ********************************************** 
       InitializingBean {

    // type = org.springframework.cloud.stream.messaging.Source
    private Class<?> type;
    
    @Autowired(required = false)
    // null
    private SharedBindingTargetRegistry sharedBindingTargetRegistry;

    // channelFactory = org.springframework.cloud.stream.binding.SubscribableChannelBindingTargetFactory
    // messageSourceFactory=org.springframework.cloud.stream.binding.MessageSourceBindingTargetFactory  
    @Autowired
	 private Map<String, BindingTargetFactory> bindingTargetFactories;

	 private Map<String, BoundTargetHolder> inputHolders = new HashMap<>();
	 private Map<String, BoundTargetHolder> outputHolders = new HashMap<>();
  
    private Object proxy;


    // ****************************************************
    // 1. Spring 回调
    // ****************************************************
    public void afterPropertiesSet() throws Exception {
        // type = org.springframework.cloud.stream.messaging.Source
        Assert.notEmpty(BindableProxyFactory.this.bindingTargetFactories, "'bindingTargetFactories' cannot be empty");
        ReflectionUtils.doWithMethods(this.type, new ReflectionUtils.MethodCallback() {
            @Override
            public void doWith(Method method) throws IllegalArgumentException {
                Input input = AnnotationUtils.findAnnotation(method, Input.class);
                if (input != null) { 
                    String name = BindingBeanDefinitionRegistryUtils.getBindingTargetName(input, method);
                    Class<?> returnType = method.getReturnType();
                    Object sharedBindingTarget = locateSharedBindingTarget(name, returnType);
                    if (sharedBindingTarget != null) {
                        BindableProxyFactory.this.inputHolders.put(name, new BoundTargetHolder(sharedBindingTarget, false));
                    } else { 
                        BindableProxyFactory.this.inputHolders.put(
                                name, 
                                new BoundTargetHolder(
                                    getBindingTargetFactory(returnType)
                                         .createInput(name), 
                                    true
                                )
                        );
                    } //end else
                } //end if
            }// end doWith
        }); //end 
  
        ReflectionUtils.doWithMethods(this.type, new ReflectionUtils.MethodCallback() {
            @Override
            public void doWith(Method method) throws IllegalArgumentException {
                    Output output = AnnotationUtils.findAnnotation(method, Output.class);
                    if (output != null) {// true
                        // output
                        String name = BindingBeanDefinitionRegistryUtils.getBindingTargetName(output, method);
                        // org.springframework.messaging.MessageChannel
                        Class<?> returnType = method.getReturnType();
                        // null
                        Object sharedBindingTarget = locateSharedBindingTarget(name, returnType);
                        if (sharedBindingTarget != null) { // false
                                BindableProxyFactory.this.outputHolders.put(name, new BoundTargetHolder(sharedBindingTarget, false));
                        } else { // true
                        
                                // *********************************************************
                                // 2. 创建MessageChannl并塞给outputHolders
                                // 调用getBindingTargetFactory(MessageChannel),创建:DirectWithAttributesChannel通道,并配置名称和出站拦截器.
                                // 将:DirectWithAttributesChannel传递给BoundTargetHolder进行包裹
                                // 将name(output)和BoundTargetHolder给Hodler住.
                                // *********************************************************
                                
                                BindableProxyFactory.this.outputHolders.put(
                                         name, 
                                         new BoundTargetHolder(
                                             // Channel
                                             getBindingTargetFactory(returnType)
                                             // output
                                                .createOutput(name), 
                                             true
                                         )
                                );
                        } //end else
                    } // end else
            } //end doWith
        }); //end 
	}// end afterPropertiesSet
 
   // 根据MessageChannel获得对应的工厂 
    private BindingTargetFactory getBindingTargetFactory(
              //org.springframework.messaging.MessageChannel
              Class<?> bindingTargetType
              ) {
        // [channelFactory]    
        List<String> candidateBindingTargetFactories = new ArrayList<>();
        
        for (Map.Entry<String, BindingTargetFactory> bindingTargetFactoryEntry : this.bindingTargetFactories.entrySet()) {
            if (bindingTargetFactoryEntry.getValue().canCreate(bindingTargetType)) {
                candidateBindingTargetFactories.add(bindingTargetFactoryEntry.getKey());
            }// end for
        } //end for
  
        if (candidateBindingTargetFactories.size() == 1) {
            // channelFactory=org.springframework.cloud.stream.binding.SubscribableChannelBindingTargetFactory
            return this.bindingTargetFactories.get(candidateBindingTargetFactories.get(0));
        } else {
                if (candidateBindingTargetFactories.size() == 0) {
                    throw new IllegalStateException("No factory found for binding target type: "
						+ bindingTargetType.getName() + " among registered factories: "
						+ StringUtils.collectionToCommaDelimitedString(bindingTargetFactories.keySet()));
                } else {
                    throw new IllegalStateException(
						"Multiple factories found for binding target type: " + bindingTargetType.getName() + ": "
								+ StringUtils.collectionToCommaDelimitedString(candidateBindingTargetFactories));
                } //end else
        }// end else
	}// end getBindingTargetFactory


    // 4. 是否为单例 
    public boolean isSingleton() {
		return true;
	  }// end isSingleton
   
   

    // 5. 获得类型
    public Class<?> getObjectType() {
          //org.springframework.cloud.stream.messaging.Source
		return this.type;
    }// end getObjectType
    
    

    // *************************************************************
    // 6. 创建动态对象
    // *************************************************************
    public synchronized Object getObject() throws Exception {
        if (this.proxy == null) {  //true  
                // 创建动态代理对象
                // proxyInterface : 为代理的接口
                // interceptor: 需要实现:org.aopalliance.intercept.Interceptor
                // org.springframework.aop.framework.ProxyFactory.ProxyFactory(Class<?> proxyInterface, Interceptorinterceptor)
                ProxyFactory factory = new ProxyFactory(this.type, this);
                this.proxy = factory.getProxy();
        } //end if
        return this.proxy;
    }// end getObject
    
    // 7.MethodInterceptor
    public synchronized Object invoke(MethodInvocation invocation) 
                throws Throwable {
        Method method = invocation.getMethod();
        
        // 7.1 先从缓存中取对象
        Object boundTarget = targetCache.get(method);
        if (boundTarget != null) {
            return boundTarget;
        } //end if

        // 获取调用的方法是否有注解@Input
        Input input = AnnotationUtils.findAnnotation(method, Input.class);
        if (input != null) {
            String name = BindingBeanDefinitionRegistryUtils.getBindingTargetName(input, method);
            boundTarget = this.inputHolders.get(name).getBoundTarget();
            targetCache.put(method, boundTarget);
            return boundTarget;
        } else {
            // 获取方法上的注解@Output
            Output output = AnnotationUtils.findAnnotation(method, Output.class);
            if (output != null) {
                // 获取方法上@Output(value="output")
                String name = BindingBeanDefinitionRegistryUtils.getBindingTargetName(output, method);
                // ************************************************
                // 在BindableProxyFactory.afterPropertiesSet中第二步
                // 中有holder住BoundTargetHolder
                // 从outputHolders中获得output
                // ************************************************
                boundTarget = this.outputHolders.get(name).getBoundTarget();
                targetCache.put(method, boundTarget);
                return boundTarget;
            }
        } // end else
        return null;
    }
    
 }
```
### (3).SubscribableChannelBindingTargetFactory
```
public class SubscribableChannelBindingTargetFactory 
       extends AbstractBindingTargetFactory<SubscribableChannel> {

    // *********************************************************      
    // 3. 创建SubscribableChannel
    // *********************************************************      
    public SubscribableChannel createOutput(String name) {
        DirectWithAttributesChannel subscribableChannel = new DirectWithAttributesChannel();
        subscribableChannel.setAttribute("type", Source.OUTPUT);
         // 委托给MessageConverterConfigurer配置输出流
         // name = output
        // 配置入站或出站消息拦截器
        this.messageChannelConfigurer.configureOutputChannel(subscribableChannel, name);
        return subscribableChannel;
	}// end createOutput
}
```
### (4).MessageConverterConfigurer
```
public class MessageConverterConfigurer 
       implements MessageChannelAndSourceConfigurer, 
                  BeanFactoryAware {
    // 4. 配置channel信息
    private void configureMessageChannel(
                       MessageChannel channel, 
                       // output
                       String channelName, 
                       boolean inbound) {
        Assert.isAssignable(AbstractMessageChannel.class, channel.getClass());
        AbstractMessageChannel messageChannel = (AbstractMessageChannel) channel;
        // 获取配置文件信息
        // output为channelName
        // spring.cloud.stream.bindings.output.destination=test-topic
        BindingProperties bindingProperties = this.bindingServiceProperties.getBindingProperties(channelName);
        String contentType = bindingProperties.getContentType();
        ProducerProperties producerProperties = bindingProperties.getProducer();
        // 如果对应的channnel的producerProperties
        if (!inbound && producerProperties != null && producerProperties.isPartitioned()) {
			messageChannel.addInterceptor(new PartitioningInterceptor(bindingProperties,
					getPartitionKeyExtractorStrategy(producerProperties),
					getPartitionSelectorStrategy(producerProperties)));
        }
        // 如果对应的channel的consumerProperties
        ConsumerProperties consumerProperties = bindingProperties.getConsumer();
        if (this.isNativeEncodingNotSet(producerProperties, consumerProperties, inbound)) {
            if (inbound) { // false
                // 入站消息拦截器
                messageChannel.addInterceptor(new InboundContentTypeEnhancingInterceptor(contentType));
            } else {
                // ************************************************************
                // 出站消息拦截器
                // ************************************************************
                messageChannel.addInterceptor(new OutboundContentTypeConvertingInterceptor(contentType, this.compositeMessageConverterFactory.getMessageConverterForAllRegistered()));
            }
        } // end if
	}// end configureMessageChannel
    
}
```
