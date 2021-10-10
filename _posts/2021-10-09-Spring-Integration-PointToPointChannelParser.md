---
layout: post
title: 'Spring Integration源码之PointToPointChannelParser(五)' 
date: 2021-10-09
author: 李新
tags:  SpringIntegration
---

### (1). 概述
这一小篇,主要剖析,SI是如何解析channel标签的.  

### (2). 查看下PointToPointChannelParser类的继承关系
```
org.springframework.beans.factory.xml.AbstractBeanDefinitionParser
	org.springframework.integration.config.xml.AbstractChannelParser
		org.springframework.integration.config.xml.PointToPointChannelParser
```
### (3). AbstractBeanDefinitionParser
AbstractBeanDefinitionParsero类的主要作用是解析XML上的id和name,把BeanDefinition向Spring容器进行注册.   

```
public abstract class AbstractBeanDefinitionParser implements BeanDefinitionParser {

	/** Constant for the "id" attribute */
	public static final String ID_ATTRIBUTE = "id";

	/** Constant for the "name" attribute */
	public static final String NAME_ATTRIBUTE = "name";


	@Override
	public final BeanDefinition parse(Element element, ParserContext parserContext) {
		AbstractBeanDefinition definition = parseInternal(element, parserContext);
		if (definition != null && !parserContext.isNested()) {
			try {
				String id = resolveId(element, definition, parserContext);
				if (!StringUtils.hasText(id)) {
					parserContext.getReaderContext().error(
							"Id is required for element '" + parserContext.getDelegate().getLocalName(element)
									+ "' when used as a top-level tag", element);
				}
				String[] aliases = null;
				if (shouldParseNameAsAliases()) {
					String name = element.getAttribute(NAME_ATTRIBUTE);
					if (StringUtils.hasLength(name)) {
						aliases = StringUtils.trimArrayElements(StringUtils.commaDelimitedListToStringArray(name));
					}
				}
				BeanDefinitionHolder holder = new BeanDefinitionHolder(definition, id, aliases);
				registerBeanDefinition(holder, parserContext.getRegistry());
				if (shouldFireEvents()) {
					BeanComponentDefinition componentDefinition = new BeanComponentDefinition(holder);
					postProcessComponentDefinition(componentDefinition);
					parserContext.registerComponent(componentDefinition);
				}
			}
			catch (BeanDefinitionStoreException ex) {
				parserContext.getReaderContext().error(ex.getMessage(), element);
				return null;
			}
		}
		return definition;
	} // end parse
	
}	
```
### (4). AbstractChannelParser
AbstractChannelParser主要是对标签interceptors进行处理.

```
// 针对标签(<interceptors></interceptors>)进行处理.
public abstract class AbstractChannelParser extends AbstractBeanDefinitionParser {

	@Override
	protected AbstractBeanDefinition parseInternal(Element element, ParserContext parserContext) {
		// 移交给子类:PointToPointChannelParser.buildBeanDefinition去构建:BeanDefinitionBuilder
		BeanDefinitionBuilder builder = this.buildBeanDefinition(element, parserContext);
		AbstractBeanDefinition beanDefinition = builder.getBeanDefinition();
		Element interceptorsElement = DomUtils.getChildElementByTagName(element, "interceptors");
		String datatypeAttr = element.getAttribute("datatype");
		String messageConverter = element.getAttribute("message-converter");
		if (!FixedSubscriberChannel.class.getName().equals(builder.getBeanDefinition().getBeanClassName())) {
			ManagedList interceptors = null;
			if (interceptorsElement != null) {
				ChannelInterceptorParser interceptorParser = new ChannelInterceptorParser();
				interceptors = interceptorParser.parseInterceptors(interceptorsElement, parserContext);
			}
			if (interceptors == null) {
				interceptors = new ManagedList();
			}
			if (StringUtils.hasText(datatypeAttr)) {
				builder.addPropertyValue("datatypes", datatypeAttr);
			}
			if (StringUtils.hasText(messageConverter)) {
				builder.addPropertyReference("messageConverter", messageConverter);
			}
			
			builder.addPropertyValue("interceptors", interceptors);
			String scopeAttr = element.getAttribute("scope");
			if (StringUtils.hasText(scopeAttr)) {
				builder.setScope(scopeAttr);
			}
		} else {
			if (interceptorsElement != null) {
				parserContext.getReaderContext().error("Cannot have interceptors when 'fixed-subscriber=\"true\"'", element);
			}
			if (StringUtils.hasText(datatypeAttr)) {
				parserContext.getReaderContext().error("Cannot have 'datatype' when 'fixed-subscriber=\"true\"'", element);
			}
			if (StringUtils.hasText(messageConverter)) {
				parserContext.getReaderContext().error("Cannot have 'message-converter' when 'fixed-subscriber=\"true\"'", element);
			}
		}
		
		beanDefinition.setSource(parserContext.extractSource(element));
		return beanDefinition;
	}

	@Override
	protected void registerBeanDefinition(BeanDefinitionHolder definition, BeanDefinitionRegistry registry) {
		String scope = definition.getBeanDefinition().getScope();
		if (!AbstractBeanDefinition.SCOPE_DEFAULT.equals(scope) && !AbstractBeanDefinition.SCOPE_SINGLETON.equals(scope) && !AbstractBeanDefinition.SCOPE_PROTOTYPE.equals(scope)) {
			definition = ScopedProxyUtils.createScopedProxy(definition, registry, false);
		}
		super.registerBeanDefinition(definition, registry);
	}

	protected abstract BeanDefinitionBuilder buildBeanDefinition(Element element, ParserContext parserContext);

}
```
### (5). PointToPointChannelParser
```
// <channel id="inputChannel"/>
// AbstractChannelParser也会完成一部份的xml解析,在这里不重要,所以,不进行跟踪了.
public class PointToPointChannelParser extends AbstractChannelParser {

	@Override
	protected BeanDefinitionBuilder buildBeanDefinition(Element element, ParserContext parserContext) {
		BeanDefinitionBuilder builder = null;
		Element queueElement = null;
		
		// 1. 解析属性:fixed-subscriber
		// <channel id="inputChannel" fixed-subscriber="false"/>
		String fixedSubscriberChannel = element.getAttribute("fixed-subscriber");
		boolean isFixedSubscriber = "true".equals(fixedSubscriberChannel.trim().toLowerCase());
		
		// 2. 解析属性:id
		// <channel id="inputChannel" fixed-subscriber="false"/>
		// configure a queue-based channel if any queue sub-element is defined
		String channel = element.getAttribute(ID_ATTRIBUTE);
		
		// 3. 解析子元素(<queue>)
		// <channel id="inputChannel" fixed-subscriber="false">
		// <queue capacity="10"/>
		// </channel>
		if ((queueElement = DomUtils.getChildElementByTagName(element, "queue")) != null) {
			builder = BeanDefinitionBuilder.genericBeanDefinition(QueueChannel.class);
			boolean hasStoreRef = this.parseStoreRef(builder, queueElement, channel, false);
			boolean hasQueueRef = this.parseQueueRef(builder, queueElement);
			if (!hasStoreRef || !hasQueueRef) {
				boolean hasCapacity = this.parseQueueCapacity(builder, queueElement);
				if (hasCapacity && hasQueueRef) {
					parserContext.getReaderContext().error(
							"The 'capacity' attribute is not allowed when providing a 'ref' to a custom queue.",
							element);
				}
				if (hasCapacity && hasStoreRef) {
					parserContext.getReaderContext().error("The 'capacity' attribute is not allowed" + " when providing a 'message-store' to a custom MessageGroupStore.", element);
				}
			}
			if (hasStoreRef && hasQueueRef) {
				parserContext.getReaderContext().error("The 'message-store' attribute is not allowed when providing a 'ref' to a custom queue.",element);
			}
		} else if ((queueElement = DomUtils.getChildElementByTagName(element, "priority-queue")) != null) {
			// 4. 解析子元素(<priority-queue>)
			// <channel id="inputChannel" fixed-subscriber="false">
			// <priority-queue capacity="10"/>
			// </channel>
			
			builder = BeanDefinitionBuilder.genericBeanDefinition(PriorityChannel.class);
			boolean hasCapacity = this.parseQueueCapacity(builder, queueElement);
			String comparatorRef = queueElement.getAttribute("comparator");
			if (StringUtils.hasText(comparatorRef)) {
				builder.addConstructorArgReference(comparatorRef);
			}
			
			if (parseStoreRef(builder, queueElement, channel, true)) {
				if (StringUtils.hasText(comparatorRef)) {
					parserContext.getReaderContext().error(
							"The 'message-store' attribute is not allowed" + " when providing a 'comparator' to a priority queue.",element);
				}
				if (hasCapacity) {
					parserContext.getReaderContext().error("The 'capacity' attribute is not allowed"
							+ " when providing a 'message-store' to a custom MessageGroupStore.", element);
				}
				builder.getRawBeanDefinition().setBeanClass(QueueChannel.class);
			}
		} else if ((queueElement = DomUtils.getChildElementByTagName(element, "rendezvous-queue")) != null) {
			
			// 5. 解析子元素(<rendezvous-queue>)
			// <channel id="inputChannel" fixed-subscriber="false">
			// <rendezvous-queue capacity="10"/>
			// </channel>
			builder = BeanDefinitionBuilder.genericBeanDefinition(RendezvousChannel.class);
		}

		// 6. 解析子元素:<dispatcher />
		Element dispatcherElement = DomUtils.getChildElementByTagName(element, "dispatcher");
		
		// 子元素dispatcher与子元素queue/priority-queue/rendezvous-queue只允许配置一个,否则,抛出异常
		// verify that a dispatcher is not provided if a queue sub-element exists
		if (queueElement != null && dispatcherElement != null) {
			parserContext.getReaderContext().error("The 'dispatcher' sub-element and any queue sub-element are mutually exclusive.", element);
			return null;
		}

		// 子元素queue/priority-queue/rendezvous-queue与属性:fixed-subscriber是排它性的,否则,会抛出异常.
		if (queueElement != null) {
			if (isFixedSubscriber) {
				parserContext.getReaderContext().error("The 'fixed-subscriber' attribute is not allowed when a <queue/> child element is present.",element);
			}
			return builder;
		}
		
		// 没有配置:dispatcher,但是,又配置了fixed-subscriber属性的情况下,生成的Bean为(FixedSubscriberChannel),否则,Bean为:DirectChannel
		if (dispatcherElement == null) { // true
			// configure the default DirectChannel with a RoundRobinLoadBalancingStrategy
			builder = BeanDefinitionBuilder.genericBeanDefinition(isFixedSubscriber ? FixedSubscriberChannel.class
					: DirectChannel.class);
		} else {
			
			// <dispatcher task-executor="" load-balancer=""/>
			// 子元素dispatcher存在内容,同时,fixed-subscriber属性也存在内容,则抛出异常.
			if (isFixedSubscriber) {
				parserContext.getReaderContext().error("The 'fixed-subscriber' attribute is not allowed" + " when a <dispatcher/> child element is present.", element);
			}
			
			// 
			// configure either an ExecutorChannel or DirectChannel based on existence of 'task-executor'
			String taskExecutor = dispatcherElement.getAttribute("task-executor");
			if (StringUtils.hasText(taskExecutor)) {
				builder = BeanDefinitionBuilder.genericBeanDefinition(ExecutorChannel.class);
				builder.addConstructorArgReference(taskExecutor);
			} else {
				builder = BeanDefinitionBuilder.genericBeanDefinition(DirectChannel.class);
			}
			
			// unless the 'load-balancer' attribute is explicitly set to 'none'
			// or 'load-balancer-ref' is explicitly configured,
			// configure the default RoundRobinLoadBalancingStrategy
			String loadBalancer = dispatcherElement.getAttribute("load-balancer");
			String loadBalancerRef = dispatcherElement.getAttribute("load-balancer-ref");
			if (StringUtils.hasText(loadBalancer) && StringUtils.hasText(loadBalancerRef)) {
				parserContext.getReaderContext().error("'load-balancer' and 'load-balancer-ref' are mutually exclusive",
						element);
			}
			
			if (StringUtils.hasText(loadBalancerRef)) {
				builder.addConstructorArgReference(loadBalancerRef);
			} else {
				if ("none".equals(loadBalancer)) {
					builder.addConstructorArgValue(null);
				}
			}
			
			IntegrationNamespaceUtils.setValueIfAttributeDefined(builder, dispatcherElement, "failover");
			IntegrationNamespaceUtils.setValueIfAttributeDefined(builder, dispatcherElement, "max-subscribers");
		}
		return builder;
	}

    // 解析 <queue capacity="10"/>
	private boolean parseQueueCapacity(BeanDefinitionBuilder builder, Element queueElement) {
		String capacity = queueElement.getAttribute("capacity");
		if (StringUtils.hasText(capacity)) {
			builder.addConstructorArgValue(capacity);
			return true;
		}
		return false;
	}

	// 解析 <queue ref=""/>
	private boolean parseQueueRef(BeanDefinitionBuilder builder, Element queueElement) {
		String queueRef = queueElement.getAttribute("ref");
		if (StringUtils.hasText(queueRef)) {
			builder.addConstructorArgReference(queueRef);
			return true;
		}
		return false;
	}
	
	// 解析 <queue message-store=""/>
	private boolean parseStoreRef(BeanDefinitionBuilder builder, Element queueElement, String channel,
			boolean priority) {
		String storeRef = queueElement.getAttribute("message-store");
		if (StringUtils.hasText(storeRef)) {
			BeanDefinitionBuilder queueBuilder = BeanDefinitionBuilder.genericBeanDefinition(MessageGroupQueue.class);
			queueBuilder.addConstructorArgReference(storeRef);
			queueBuilder.addConstructorArgValue(new TypedStringValue(storeRef).getValue() + ":" + channel);
			queueBuilder.addPropertyValue("priority", priority);
			parseQueueCapacity(queueBuilder, queueElement);
			builder.addConstructorArgValue(queueBuilder.getBeanDefinition());
			return true;
		}
		return false;
	}
}
```
### (6). MessageChannel
分析上面的源码,我们能得出一个结论就是,最终channel标签会被解析成如下对象,默认MessageChannel的实现为:DirectChannel:  
 
```
 QueueChannel                   : 消息被发布到QueueChannel后被存储到一个队列中,直到消息被消费者以先进先出(FIFO)的方式拉取.如果有多个消费者,他们中只有一个能收到消息.
 PriorityChannel                : 与QueueChannel类似,但是与FIFO方式不同,消息被冠以priority的消费者拉取.
 RendezvousChannel              : 发送者阻塞通道,直到消费者接收这个消息类.
 FixedSubscriberChannel         : 固定的消费者通道
 DirectChannel                  : 通过在与发送方相同的线程中调用消费者来将消息发送给单个消费者,此通道类型允许事务跨越通道.
 ExecutorChannel                : 与DirectChannel类似,但是消息分派是通过TaskExecutor进行的,在与发送方不同的线程中进行,此通道类型不支持事务跨通道.   
 ```
### (7). DirectChannel类的继承关系
```
org.springframework.integration.context.IntegrationObjectSupport
	org.springframework.integration.channel.AbstractMessageChannel
		org.springframework.integration.channel.AbstractSubscribableChannel
			org.springframework.integration.channel.DirectChannel
```
### (8). DirectChannel核心代码
```
public class DirectChannel extends AbstractSubscribableChannel {
	// ***************************************************************************
	// 创建一个单播的分发器
	// ***************************************************************************
	private final UnicastingDispatcher dispatcher = new UnicastingDispatcher();
	
	protected UnicastingDispatcher getDispatcher() {
		return this.dispatcher;
	}
} // end DirectChannel

public abstract class AbstractSubscribableChannel extends AbstractMessageChannel
		implements SubscribableChannel, SubscribableChannelManagement {
    
	// ***************************************************************************
	// 订阅
	// 在:service-activator内部,会找到:input-channel,调用该方法,注册订阅者.
	// <service-activator input-channel="inputChannel" output-channel="outputChannel" ref="helloService" method="sayHello"/>
	// ***************************************************************************
	@Override
	public boolean subscribe(MessageHandler handler) {
		MessageDispatcher dispatcher = getRequiredDispatcher();
		boolean added = dispatcher.addHandler(handler);
		this.adjustCounterIfNecessary(dispatcher, added ? 1 : 0);
		return added;
	}

	// ***************************************************************************
	// 取消订阅
	// ***************************************************************************
	@Override
	public boolean unsubscribe(MessageHandler handle) {
		MessageDispatcher dispatcher = getRequiredDispatcher();
		boolean removed = dispatcher.removeHandler(handle);
		this.adjustCounterIfNecessary(dispatcher, removed ? -1 : 0);
		return removed;
	}
	
}// end AbstractSubscribableChannel
```
### (9). DirectChannel类结构图
+ AbstractMessageChannel负责消息的发送功能.    
+ AbstractSubscribableChannel负责消息的订阅功能.  
+ UnicastingDispatcher负责消息的单播功能.   


!["DirectChannel类结构图"](/assets/spring-integration/imgs/DirectChannel-Diagram.jpg)
### (10). 总结
通过对PointToPointChannelParser源码的剖析,能知道:channel是具有"发送"和"订阅消息"功能的,所以,为什么会定义为Channel,而不是Product或Consumer的原因就在于此.      