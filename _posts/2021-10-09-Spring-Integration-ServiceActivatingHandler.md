---
layout: post
title: 'Spring Integration源码之ServiceActivatingHandler(八)' 
date: 2021-10-09
author: 李新
tags:  SpringIntegration
---

### (1). 概述
在这一小篇,主要剖析:ServiceActivatingHandler.

### (2). 先看下ServiceActivatingHandler类的继承关系
```
org.springframework.integration.context.IntegrationObjectSupport
	org.springframework.integration.handler.AbstractMessageHandler
		org.springframework.integration.handler.AbstractMessageProducingHandler
			org.springframework.integration.handler.AbstractReplyProducingMessageHandler
				org.springframework.integration.handler.ServiceActivatingHandler
```
### (3). 看下ServiceActivatingHandler类结构图
+ 包裹业务对象,转化成:MessageProcessor.  
+ 实现了MessageHandler接口,用于接收消息.   
+ 内部持有:MessageChannel,用于输出消息.   

!["ServiceActivatingHandler"](/assets/spring-integration/imgs/ServiceActivatingHandler-Diagram.jpg)

### (4). AbstractReplyProducingMessageHandler
> 为什么入口是:AbstractReplyProducingMessageHandler.onInit方法?因为:IntegrationObjectSupport方法,实现了:InitializingBean.Spring初始化时会回调:afterPropertiesSet方法.

```
// 从代码分析来看:afterPropertiesSet仅仅是做了一些初始化的工作.

public abstract class IntegrationObjectSupport implements BeanNameAware, NamedComponent,
		ApplicationContextAware, BeanFactoryAware, InitializingBean, ExpressionCapable {
    
	// *************************************************************************
	// 1. 由于实现了:InitializingBean,所以,Spring会回调该方法.
	// *************************************************************************
	public final void afterPropertiesSet() {
		this.integrationProperties = IntegrationContextUtils.getIntegrationProperties(this.beanFactory);
		try {
			if (this.messageBuilderFactory == null) {
				if (this.beanFactory != null) {
					this.messageBuilderFactory = IntegrationUtils.getMessageBuilderFactory(this.beanFactory);
				}
				else {
					this.messageBuilderFactory = new DefaultMessageBuilderFactory();
				}
			}
			// *************************************************************************
			// 2.委托给孙类:AbstractReplyProducingMessageHandler.onInit,因为:AbstractReplyProducingMessageHandler重写了该方法
			// *************************************************************************
			this.onInit();
		}
		catch (Exception e) {
			if (e instanceof RuntimeException) {
				throw (RuntimeException) e;
			}
			throw new BeanInitializationException("failed to initialize", e);
		}
	} // end afterPropertiesSet
	
}

public abstract class AbstractMessageHandler extends IntegrationObjectSupport implements MessageHandler,
		MessageHandlerMetrics, ConfigurableMetricsAware<AbstractMessageHandlerMetrics>, TrackableComponent, Orderable {
    
	// *************************************************************************
	// 6. 初始化
	// *************************************************************************
    protected void onInit() throws Exception {
		if (this.statsEnabled) {
			this.handlerMetrics.setFullStatsEnabled(true);
		}
	}
}

public abstract class AbstractMessageProducingHandler extends AbstractMessageHandler
		implements MessageProducer, HeaderPropagationAware {
    
	private final Set<String> notPropagatedHeaders = new HashSet<String>();
	protected final MessagingTemplate messagingTemplate = new MessagingTemplate();
	private volatile MessageChannel outputChannel;
	private volatile String outputChannelName;
	private volatile boolean async;
	private boolean selectiveHeaderPropagation;
	
	
	protected void onInit() throws Exception {
		// *************************************************************************
		// 5. 先调用父类AbstractMessageHandler.onInit()
		// *************************************************************************
		super.onInit();
		Assert.state(!(this.outputChannelName != null && this.outputChannel != null), //NOSONAR (inconsistent sync)
				"'outputChannelName' and 'outputChannel' are mutually exclusive.");
		if (getBeanFactory() != null) {
			// 配置BeanFactory
			this.messagingTemplate.setBeanFactory(getBeanFactory());
		}
		// 配置:BeanFactoryChannelResolver
		this.messagingTemplate.setDestinationResolver(getChannelResolver());
	}
} // end AbstractMessageProducingHandler


public abstract class AbstractReplyProducingMessageHandler 
		extends AbstractMessageProducingHandler
		implements BeanClassLoaderAware {
    
	// *************************************************************************
	// 3. IntegrationObjectSupport触发onInit
	// *************************************************************************
	protected final void onInit() throws Exception {
		// *************************************************************************
		// 4. 先调用父类AbstractMessageProducingHandler.onInit()
		// *************************************************************************
		
		super.onInit();
		
		if (!CollectionUtils.isEmpty(this.adviceChain)) {
			ProxyFactory proxyFactory = new ProxyFactory(new AdvisedRequestHandler());
			boolean advised = false;
			for (Advice advice : this.adviceChain) {
				if (!(advice instanceof HandleMessageAdvice)) {
					proxyFactory.addAdvice(advice);
					advised = true;
				}
			}
			if (advised) {
				this.advisedRequestHandler = (RequestHandler) proxyFactory.getProxy(this.beanClassLoader);
			}
		}
		
		// *************************************************************************
		// 7. 回调子类ServiceActivatingHandler.doInit
		// *************************************************************************
		doInit();
	} // end onInit
} // end AbstractReplyProducingMessageHandler


public class ServiceActivatingHandler extends AbstractReplyProducingMessageHandler implements Lifecycle {
	
	// 包裹业务对象
	private final MessageProcessor<?> processor;
	
	
	// *************************************************************************
	// 8. ServiceActivatingHandler处理初始化
	// *************************************************************************
	protected void doInit() {
		// 为MessageProcessor配置:ConversionService
		if (this.processor instanceof AbstractMessageProcessor) {
			((AbstractMessageProcessor<?>) this.processor).setConversionService(this.getConversionService());
		}
		
		// 为MessageProcessor配置:BeanFactory
		if (this.processor instanceof BeanFactoryAware && this.getBeanFactory() != null) {
			((BeanFactoryAware) this.processor).setBeanFactory(this.getBeanFactory());
		}
	} // end doInit
}
```
### (5). AbstractMessageHandler.handleMessage
```
public abstract class AbstractMessageHandler extends IntegrationObjectSupport implements MessageHandler,
		MessageHandlerMetrics, ConfigurableMetricsAware<AbstractMessageHandlerMetrics>, TrackableComponent, Orderable {

	public void handleMessage(Message<?> message) {
		// ... ...
		MetricsContext start = null;
		boolean countsEnabled = this.countsEnabled;
		AbstractMessageHandlerMetrics handlerMetrics = this.handlerMetrics;
		try {
			if (message != null && this.shouldTrack) { // 是否为track,track对message又进行了处理.
				message = MessageHistory.write(message, this, this.getMessageBuilderFactory());
			}
			
			if (countsEnabled) {
				start = handlerMetrics.beforeHandle();
			}
			
			// ***********************************************************************
			// 委托给了孙类:AbstractReplyProducingMessageHandler进行处理
			// ***********************************************************************
			this.handleMessageInternal(message);
			
			if (countsEnabled) {
				handlerMetrics.afterHandle(start, true);
			}
		} catch (Exception e) {
			if (countsEnabled) {
				handlerMetrics.afterHandle(start, false);
			}
			
			if (e instanceof MessagingException) {
				throw (MessagingException) e;
			}
			throw new MessageHandlingException(message, "error occurred in message handler [" + this + "]", e);
		}
	} // end handleMessage

}
```
### (6). AbstractReplyProducingMessageHandler.handleMessageInternal
```
public abstract class AbstractReplyProducingMessageHandler extends AbstractMessageProducingHandler
		implements BeanClassLoaderAware {

    protected final void handleMessageInternal(Message<?> message) {
		Object result;
		if (this.advisedRequestHandler == null) {
			result = handleRequestMessage(message);
		} else {
			result = doInvokeAdvisedRequestHandler(message);
		}
		
		if (result != null) {
			// ****************************************************************
			// 委托给了父类(AbstractMessageProducingHandler.sendOutputs)
			// ****************************************************************
			sendOutputs(result, message);
		} else if (this.requiresReply && !isAsync()) {
			throw new ReplyRequiredException(message, "No reply produced by handler '" + getComponentName() + "', and its 'requiresReply' property is set to true.");
		} else if (!isAsync() && logger.isDebugEnabled()) {
			logger.debug("handler '" + this + "' produced no reply for request Message: " + message);
		}
	} // end handleMessageInternal
}
```
### (7). AbstractMessageProducingHandler.sendOutputs
```
protected void sendOutputs(Object result, Message<?> requestMessage) {
	if (result instanceof Iterable<?> && shouldSplitOutput((Iterable<?>) result)) {
		for (Object o : (Iterable<?>) result) {
			// ***********************************************************
			// 1.委托给了本地
			// ***********************************************************
			this.produceOutput(o, requestMessage);
		}
	} else if (result != null) {
		// ***********************************************************
		// 1. 委托给了本地
		// ***********************************************************
		this.produceOutput(result, requestMessage);
	}
} // end sendOutputs


// ***************************************************************************
// 往
// ***************************************************************************
protected void produceOutput(Object reply, final Message<?> requestMessage) {
	final MessageHeaders requestHeaders = requestMessage.getHeaders();

	Object replyChannel = null;
	if (getOutputChannel() == null) { // false
		Map<?, ?> routingSlipHeader = requestHeaders.get(IntegrationMessageHeaderAccessor.ROUTING_SLIP, Map.class);
		if (routingSlipHeader != null) {
			Assert.isTrue(routingSlipHeader.size() == 1, "The RoutingSlip header value must be a SingletonMap");
			Object key = routingSlipHeader.keySet().iterator().next();
			Object value = routingSlipHeader.values().iterator().next();
			Assert.isInstanceOf(List.class, key, "The RoutingSlip key must be List");
			Assert.isInstanceOf(Integer.class, value, "The RoutingSlip value must be Integer");
			List<?> routingSlip = (List<?>) key;
			AtomicInteger routingSlipIndex = new AtomicInteger((Integer) value);
			replyChannel = getOutputChannelFromRoutingSlip(reply, requestMessage, routingSlip, routingSlipIndex);
			if (replyChannel != null) {
				//TODO Migrate to the SF MessageBuilder
				AbstractIntegrationMessageBuilder<?> builder = null;
				if (reply instanceof Message) {
					builder = this.getMessageBuilderFactory().fromMessage((Message<?>) reply);
				}
				else if (reply instanceof AbstractIntegrationMessageBuilder) {
					builder = (AbstractIntegrationMessageBuilder<?>) reply;
				}
				else {
					builder = this.getMessageBuilderFactory().withPayload(reply);
				}
				builder.setHeader(IntegrationMessageHeaderAccessor.ROUTING_SLIP,
						Collections.singletonMap(routingSlip, routingSlipIndex.get()));
				reply = builder;
			}
		}

		if (replyChannel == null) {
			replyChannel = requestHeaders.getReplyChannel();
		}
	}

	if (this.async && reply instanceof ListenableFuture<?>) {
		ListenableFuture<?> future = (ListenableFuture<?>) reply;
		final Object theReplyChannel = replyChannel;
		future.addCallback(new ListenableFutureCallback<Object>() {

			@Override
			public void onSuccess(Object result) {
				Message<?> replyMessage = null;
				try {
					replyMessage = createOutputMessage(result, requestHeaders);
					sendOutput(replyMessage, theReplyChannel, false);
				}
				catch (Exception e) {
					Exception exceptionToLogAndSend = e;
					if (!(e instanceof MessagingException)) {
						exceptionToLogAndSend = new MessageHandlingException(requestMessage, e);
						if (replyMessage != null) {
							exceptionToLogAndSend = new MessagingException(replyMessage, exceptionToLogAndSend);
						}
					}
					logger.error("Failed to send async reply: " + result.toString(), exceptionToLogAndSend);
					onFailure(exceptionToLogAndSend);
				}
			}

			@Override
			public void onFailure(Throwable ex) {
				sendErrorMessage(requestMessage, ex);
			}
		});
	} else {
		// 
		sendOutput(createOutputMessage(reply, requestHeaders), replyChannel, false);
	}
} // end produceOutput

// 3. 根据channel名称(outputChannelName),从Spring容器里获得:MessageChannel对象
public MessageChannel getOutputChannel() {
	if (this.outputChannelName != null) {
		synchronized (this) {
			if (this.outputChannelName != null) {
				this.outputChannel = getChannelResolver().resolveDestination(this.outputChannelName);
				this.outputChannelName = null;
			}
		}
	}
	return this.outputChannel;
} // end getOutputChannel


// 4. 最终发送消息的方法
protected void sendOutput(Object output, Object replyChannel, boolean useArgChannel) {
	// 从Spring容器中,获得:MessageChannel
	MessageChannel outputChannel = getOutputChannel();
	if (!useArgChannel && outputChannel != null) {
		replyChannel = outputChannel;
	}
	
	if (replyChannel == null) {
		throw new DestinationResolutionException("no output-channel or replyChannel header available");
	}

	if (replyChannel instanceof MessageChannel) {
		if (output instanceof Message<?>) {
			this.messagingTemplate.send((MessageChannel) replyChannel, (Message<?>) output);
		} else {
			this.messagingTemplate.convertAndSend((MessageChannel) replyChannel, output);
		}
	} else if (replyChannel instanceof String) {
		if (output instanceof Message<?>) {
			// **********************************************************************************
			// 通过MessagingTemplate发送消息
			// **********************************************************************************
			this.messagingTemplate.send((String) replyChannel, (Message<?>) output);
		} else {
			// **********************************************************************************
			// 通过MessagingTemplate发送消息
			// **********************************************************************************
			this.messagingTemplate.convertAndSend((String) replyChannel, output);
		}
	} else {
		throw new MessagingException("replyChannel must be a MessageChannel or String");
	}
} // end sendOutput
```
### (8). 总结
ServiceActivatingHandler的目的是被动接受消息,并把消息放MessageChannel(outputChannel)进行发送.   
