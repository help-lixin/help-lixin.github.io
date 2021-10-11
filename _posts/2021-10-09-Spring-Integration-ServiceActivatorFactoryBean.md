---
layout: post
title: 'Spring Integration源码之ServiceActivatorFactoryBean(七)' 
date: 2021-10-09
author: 李新
tags:  SpringIntegration
---

### (1). 概述
前面,通过源码剖析,SI会委托给ServiceActivatorParser解析service-activator标签,最终会转换成业务模型:ServiceActivatorFactoryBean,在这一小节,主要是剖析:ServiceActivatorFactoryBean.   

### (2). 先看下ServiceActivatorFactoryBean类的关系图
```
org.springframework.integration.config.AbstractSimpleMessageHandlerFactoryBean
	org.springframework.integration.config.AbstractStandardMessageHandlerFactoryBean
		org.springframework.integration.config.ServiceActivatorFactoryBean
```
### (3). AbstractSimpleMessageHandlerFactoryBean
```
public abstract class AbstractSimpleMessageHandlerFactoryBean<H extends MessageHandler>
		implements 
		// **********************************************************************************
		// FactoryBean固名思义,就是产生Bean的工厂,所以,重点要关注这个类的:getObject方法.
		// **********************************************************************************
		FactoryBean<MessageHandler>, 
		ApplicationContextAware, 
		BeanFactoryAware, 
		BeanNameAware, 
		ApplicationEventPublisherAware {
}			
```
### (4). AbstractSimpleMessageHandlerFactoryBean
```
public abstract class AbstractSimpleMessageHandlerFactoryBean<H extends MessageHandler>
		implements FactoryBean<MessageHandler>, ApplicationContextAware, BeanFactoryAware, BeanNameAware,
		ApplicationEventPublisherAware {
	
	private volatile H handler;

	@Override
	public H getObject() throws Exception {
		if (this.handler == null) {
			// *******************************************************************************************
			// 1. 委托给了:createHandlerInternal
			// *******************************************************************************************
			this.handler = this.createHandlerInternal();
			Assert.notNull(this.handler, "failed to create MessageHandler");
		}
		return this.handler;
	} // end getObject
	
	protected final H createHandlerInternal() {
		synchronized (this.initializationMonitor) {
			if (this.initialized) {
				// There was a problem when this method was called already
				return null;
			}
			
			// *******************************************************************************************
			// 2. 委托给了子类(AbstractStandardMessageHandlerFactoryBean)的createHandler
			// *******************************************************************************************
			this.handler = createHandler();
			
			// 余下的信息,都是配置Spring的一些钩子对象(ApplicationContextAware/BeanNameAware/ApplicationEventPublisherAware/MessageProducer/IntegrationObjectSupport/)
			if (this.handler instanceof ApplicationContextAware && this.applicationContext != null) {
				((ApplicationContextAware) this.handler).setApplicationContext(this.applicationContext);
			}
			
			if (this.handler instanceof BeanFactoryAware && getBeanFactory() != null) {
				((BeanFactoryAware) this.handler).setBeanFactory(getBeanFactory());
			}
			
			if (this.handler instanceof BeanNameAware && this.beanName != null) {
				((BeanNameAware) this.handler).setBeanName(this.beanName);
			}
			
			if (this.handler instanceof ApplicationEventPublisherAware && this.applicationEventPublisher != null) {
				((ApplicationEventPublisherAware) this.handler)
						.setApplicationEventPublisher(this.applicationEventPublisher);
			}
			
<<<<<<< HEAD
			// ********************************************************************************
			// 设置setOutputChannel
			// ********************************************************************************
=======
			// ***************************************************************************
			// 设置:outputChannel
			// ***************************************************************************
>>>>>>> b6ab82f117acbb232196bc1220b46d22870d945e
			if (this.handler instanceof MessageProducer && this.outputChannel != null) {
				((MessageProducer) this.handler).setOutputChannel(this.outputChannel);
			}
			
			Object actualHandler = extractTarget(this.handler);
			if (actualHandler == null) {
				actualHandler = this.handler;
			}
			
			if (actualHandler instanceof IntegrationObjectSupport) {
				if (this.componentName != null) {
					((IntegrationObjectSupport) actualHandler).setComponentName(this.componentName);
				}
				if (this.channelResolver != null) {
					((IntegrationObjectSupport) actualHandler).setChannelResolver(this.channelResolver);
				}
			}
			
			// 好像还有责任链的功能
			if (!CollectionUtils.isEmpty(this.adviceChain)) {
				if (actualHandler instanceof AbstractReplyProducingMessageHandler) {
					((AbstractReplyProducingMessageHandler) actualHandler).setAdviceChain(this.adviceChain);
				}
				else if (this.logger.isDebugEnabled()) {
					String name = this.componentName;
					if (name == null && actualHandler instanceof NamedComponent) {
						name = ((NamedComponent) actualHandler).getComponentName();
					}
					this.logger.debug("adviceChain can only be set on an AbstractReplyProducingMessageHandler"
						+ (name == null ? "" : (", " + name)) + ".");
				}
			}
			
			if (this.async != null) {
				if (actualHandler instanceof AbstractReplyProducingMessageHandler) {
					((AbstractReplyProducingMessageHandler) actualHandler)
							.setAsync(this.async);
				}
			}
			
			if (this.handler instanceof Orderable && this.order != null) {
				((Orderable) this.handler).setOrder(this.order);
			}
			this.initialized = true;
		}
		
		
		if (this.handler instanceof InitializingBean) {
			try {
				// ********************************************************************************
				// 手动调用:afterPropertiesSet
				// ********************************************************************************
				((InitializingBean) this.handler).afterPropertiesSet();
			}
			catch (Exception e) {
				throw new BeanInitializationException("failed to initialize MessageHandler", e);
			}
		}
		
		return this.handler;
	} // end createHandlerInternal
}
```
### (5). AbstractStandardMessageHandlerFactoryBean
```
public abstract class AbstractStandardMessageHandlerFactoryBean
		extends AbstractSimpleMessageHandlerFactoryBean<MessageHandler> {
    
	private static final ExpressionParser expressionParser = new SpelExpressionParser(new SpelParserConfiguration(true,
				true));
	
	private static final Set<MessageHandler> referencedReplyProducers = new HashSet<MessageHandler>();
	
	// *******************************************************************************************
	// 目标对象(org.springframework.integration.samples.helloworld.HelloService)
	// *******************************************************************************************
	private volatile Object targetObject;
	
	// *******************************************************************************************
	// 目标方法(sayHello)
	// *******************************************************************************************
	private volatile String targetMethodName;
	
	private volatile Expression expression;
	
	// *******************************************************************************************
	// 创建:MessageHandler
	// *******************************************************************************************
	protected MessageHandler createHandler() {
		MessageHandler handler;
		
		// 1. 验证targetMethodName是否不为字符串
		if (this.targetObject == null) {
			Assert.isTrue(!StringUtils.hasText(this.targetMethodName), "The target method is only allowed when a target object (ref or inner bean) is also provided.");
		}
		
	
		if (this.targetObject != null) { // 2. 针对targetObject包装成:MessageHandler
			Assert.state(this.expression == null, "The 'targetObject' and 'expression' properties are mutually exclusive.");
			
			// 3. 判断targetObject是否为:AbstractMessageProducingHandler,我这里为null
			AbstractMessageProducingHandler actualHandler = this.extractTypeIfPossible(this.targetObject, AbstractMessageProducingHandler.class);
			
			// 4. targetIsDirectReplyProducingHandler = false
			boolean targetIsDirectReplyProducingHandler = actualHandler != null && this.canBeUsedDirect(actualHandler) && this.methodIsHandleMessageOrEmpty(this.targetMethodName);
			
			// 5. 判断targetObject(HelloService) 是否为: MessageProcessor
			if (this.targetObject instanceof MessageProcessor<?>) { //  false
				handler = this.createMessageProcessingHandler((MessageProcessor<?>) this.targetObject);
			} else if (targetIsDirectReplyProducingHandler) {  // false
				if (logger.isDebugEnabled()) {
					logger.debug("Wiring handler (" + this.targetObject + ") directly into endpoint");
				}
				
				this.checkReuse(actualHandler);
				this.postProcessReplyProducer(actualHandler);
				handler = (MessageHandler) this.targetObject;
			} else {
				// ***********************************************************************************
				// 6. 委托给子类(ServiceActivatorFactoryBean)创建:MessageHandler
				// ***********************************************************************************
				handler = this.createMethodInvokingHandler(this.targetObject, this.targetMethodName);
			}
		} else if (this.expression != null) { 
			// 针对expression包装成:MessageHandler
			handler = this.createExpressionEvaluatingHandler(this.expression);
		} else {  
			// 既没指定:targetObject和expression,则,创建默认的:MessageHandler
			handler = this.createDefaultHandler();
		}
		return handler;
	} // end createHandler
				
}
```
### (6). ServiceActivatorFactoryBean
```
public class ServiceActivatorFactoryBean extends AbstractStandardMessageHandlerFactoryBean {
	
	// *******************************************************************************************
	// 
	// *******************************************************************************************
	protected MessageHandler createMethodInvokingHandler(Object targetObject, String targetMethodName) {
		MessageHandler handler = null;
		
		// 1. 如果targetObject为MessageHandler并且,targetMethodName为:handleMessage的话,则,返回:MessageHandler
		handler = createDirectHandlerIfPossible(targetObject, targetMethodName);
		
		if (handler == null) { // true
			handler = configureHandler(
					//  判断:targetMethodName(sayHello)是否为字符串.
					StringUtils.hasText(targetMethodName)
					// ********************************************************************************************
					// 2. 创建:ServiceActivatingHandler对象,包裹着:targetObject与targetMethodName
					// ********************************************************************************************
					? new ServiceActivatingHandler(targetObject, targetMethodName)
					// 
					: new ServiceActivatingHandler(targetObject));
		}
		return handler;
	}// end createMethodInvokingHandler
		
}
```
### (7). 总结
ServiceActivatorFactoryBean的工厂方法(getObject),最终是创建了:ServiceActivatingHandler(属于MessageHandler的实现)对象包裹我们的业务对象(HelloService)和方法(sayHello).   