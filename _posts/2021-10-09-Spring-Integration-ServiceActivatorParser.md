---
layout: post
title: 'Spring Integration源码之ServiceActivatorParser(六)' 
date: 2021-10-09
author: 李新
tags:  SpringIntegration
---

### (1). 概述
这一小篇,主要剖析,SI是如何解析service-activator标签的

### (2). xml内容如下
```
<beans:bean id="helloService" class="org.springframework.integration.samples.helloworld.HelloService"/>

<service-activator input-channel="inputChannel"
	                   output-channel="outputChannel"
	                   ref="helloService"
	                   method="sayHello"/>
```
### (3). ServiceActivatorParser继承关系图
```
org.springframework.beans.factory.xml.AbstractBeanDefinitionParser
	org.springframework.integration.config.xml.AbstractConsumerEndpointParser             # 在SI的眼里<service-activator>是Consumer
		org.springframework.integration.config.xml.AbstractDelegatingConsumerEndpointParser
			org.springframework.integration.config.xml.ServiceActivatorParser
```
### (4). AbstractConsumerEndpointParser
```
public abstract class AbstractConsumerEndpointParser extends AbstractBeanDefinitionParser {
	protected static final String REF_ATTRIBUTE = "ref";
	protected static final String METHOD_ATTRIBUTE = "method";
	protected static final String EXPRESSION_ATTRIBUTE = "expression";
	
	protected abstract BeanDefinitionBuilder parseHandler(Element element, ParserContext parserContext);
	
	protected String getInputChannelAttributeName() {
		return "input-channel";
	}
	
	@Override
	protected final AbstractBeanDefinition parseInternal(Element element, ParserContext parserContext) {
		// *****************************************************************************
		// 1.委托给子类:AbstractDelegatingConsumerEndpointParser.parseHandler
		// 最终返回的BeanDefinitionBuilder包裹着:ServiceActivatorFactoryBean
		// 而,ServiceActivatorFactoryBean是一个工厂方法,每次都会创建一个新的:MessageHandler
		// 也就是说:<service-activator>标签会解析出一个部信息,转化成:MessageHandler
		// *****************************************************************************
		BeanDefinitionBuilder handlerBuilder = this.parseHandler(element, parserContext);
		
		// 为ServiceActivatorFactoryBean配置outputChannel的引用
		IntegrationNamespaceUtils.setReferenceIfAttributeDefined(handlerBuilder, element, "output-channel");
		// 为ServiceActivatorFactoryBean配置order
		IntegrationNamespaceUtils.setValueIfAttributeDefined(handlerBuilder, element, "order");

		// 配置:request-handler-advice-chain
		Element adviceChainElement = DomUtils.getChildElementByTagName(element, IntegrationNamespaceUtils.REQUEST_HANDLER_ADVICE_CHAIN);
		IntegrationNamespaceUtils.configureAndSetAdviceChainIfPresent(adviceChainElement, null, handlerBuilder.getRawBeanDefinition(), parserContext);
		
		// handlerBeanDefinition = org.springframework.integration.config.ServiceActivatorFactoryBean
		AbstractBeanDefinition handlerBeanDefinition = handlerBuilder.getBeanDefinition();
		// input-channel
		String inputChannelAttributeName = this.getInputChannelAttributeName();
		//true
		boolean hasInputChannelAttribute = element.hasAttribute(inputChannelAttributeName);
		if (parserContext.isNested()) { // false
			String elementDescription = IntegrationNamespaceUtils.createElementDescription(element);
			if (hasInputChannelAttribute) {
				parserContext.getReaderContext().error("The '" + inputChannelAttributeName
						+ "' attribute isn't allowed for a nested (e.g. inside a <chain/>) endpoint element: "
						+ elementDescription + ".", element);
			}
			if (!replyChannelInChainAllowed(element)) {
				if (StringUtils.hasText(element.getAttribute("reply-channel"))) {
					parserContext.getReaderContext().error("The 'reply-channel' attribute isn't"
							+ " allowed for a nested (e.g. inside a <chain/>) outbound gateway element: "
							+ elementDescription + ".", element);
				}
			}
			return handlerBeanDefinition;
		} else {
			if (!hasInputChannelAttribute) { // false
				String elementDescription = IntegrationNamespaceUtils.createElementDescription(element);
				parserContext.getReaderContext().error("The '" + inputChannelAttributeName
						+ "' attribute is required for the top-level endpoint element: "
						+ elementDescription + ".", element);
			}
		}

		// ********************************************************************************
		// 2. 声明Bean:ConsumerEndpointFactoryBean这是核心类
		// ********************************************************************************
		BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition(ConsumerEndpointFactoryBean.class);
		
		// handlerBeanName = org.springframework.integration.config.ServiceActivatorFactoryBean#0
		String handlerBeanName = BeanDefinitionReaderUtils.generateBeanName(handlerBeanDefinition, parserContext.getRegistry());
		// handlerAlias = null
		String[] handlerAlias = IntegrationNamespaceUtils.generateAlias(element);
		
		// ********************************************************************************
		// 3. 向Spring容器中注册bean,
		//    id = org.springframework.integration.config.ServiceActivatorFactoryBean#0
		//    class = org.springframework.integration.config.ServiceActivatorFactoryBean)
		// ********************************************************************************
		parserContext.registerBeanComponent(new BeanComponentDefinition(handlerBeanDefinition, handlerBeanName, handlerAlias));
		
		// 为ConsumerEndpointFactoryBean配置:handler = org.springframework.integration.config.ServiceActivatorFactoryBean#0
		builder.addPropertyReference("handler", handlerBeanName);
		
		// inputChannel
		String inputChannelName = element.getAttribute(inputChannelAttributeName);
		
		// 判断bean定义(inputChannel)在spring容器中是否存在.
		if (!parserContext.getRegistry().containsBeanDefinition(inputChannelName)) {  // false
			// ntegrationContextUtils.AUTO_CREATE_CHANNEL_CANDIDATES_BEAN_NAME = $autoCreateChannelCandidates
			if (parserContext.getRegistry().containsBeanDefinition(IntegrationContextUtils.AUTO_CREATE_CHANNEL_CANDIDATES_BEAN_NAME)) {
				BeanDefinition channelRegistry = parserContext.getRegistry().getBeanDefinition(IntegrationContextUtils.AUTO_CREATE_CHANNEL_CANDIDATES_BEAN_NAME);
				ConstructorArgumentValues caValues = channelRegistry.getConstructorArgumentValues();
				ValueHolder vh = caValues.getArgumentValue(0, Collection.class);
				if (vh == null) { //although it should never happen if it does we can fix it
					caValues.addIndexedArgumentValue(0, new ManagedSet<String>());
				}

				@SuppressWarnings("unchecked")
				Collection<String> channelCandidateNames = (Collection<String>) caValues.getArgumentValue(0, Collection.class).getValue();
				channelCandidateNames.add(inputChannelName);
			} else {
				parserContext.getReaderContext().error("Failed to locate '" +
						IntegrationContextUtils.AUTO_CREATE_CHANNEL_CANDIDATES_BEAN_NAME + "'", parserContext.getRegistry());
			}
		}
		
		
		IntegrationNamespaceUtils.checkAndConfigureFixedSubscriberChannel(element, parserContext, inputChannelName,handlerBeanName);
		
		// 为ConsumerEndpointFactoryBean配置:inputChannelName=inputChannel
		builder.addPropertyValue("inputChannelName", inputChannelName);
		
		// 针对子节点<poller/>处理
		List<Element> pollerElementList = DomUtils.getChildElementsByTagName(element, "poller");
		if (!CollectionUtils.isEmpty(pollerElementList)) {
			if (pollerElementList.size() != 1) {
				parserContext.getReaderContext().error(
						"at most one poller element may be configured for an endpoint", element);
			}
			IntegrationNamespaceUtils.configurePollerMetadata(pollerElementList.get(0), builder, parserContext);
		}
		
		// 为ConsumerEndpointFactoryBean配置:autoStartup
		IntegrationNamespaceUtils.setValueIfAttributeDefined(builder, element, IntegrationNamespaceUtils.AUTO_STARTUP);
		// 为ConsumerEndpointFactoryBean配置:phase
		IntegrationNamespaceUtils.setValueIfAttributeDefined(builder, element, IntegrationNamespaceUtils.PHASE);
		
		String role = element.getAttribute(IntegrationNamespaceUtils.ROLE);
		if (StringUtils.hasText(role)) {
			if (!StringUtils.hasText(element.getAttribute(ID_ATTRIBUTE))) {
				parserContext.getReaderContext().error("When using 'role', 'id' is required", element);
			}
			IntegrationNamespaceUtils.putLifecycleInRole(role, element.getAttribute(ID_ATTRIBUTE), parserContext);
		}
		
		// 获得ConsumerEndpointFactoryBean定义信息
		AbstractBeanDefinition beanDefinition = builder.getBeanDefinition();
		// 解析出bean名称(org.springframework.integration.config.ConsumerEndpointFactoryBean#0)
		String beanName = this.resolveId(element, beanDefinition, parserContext);
		
		// ********************************************************************************
		// 向Spring容器中,注册Bean
		//  id     = org.springframework.integration.config.ConsumerEndpointFactoryBean#0
		//  class  = org.springframework.integration.config.ConsumerEndpointFactoryBean
		// ********************************************************************************
		parserContext.registerBeanComponent(new BeanComponentDefinition(beanDefinition, beanName));
		return null;
	} // end parseInternal
}
```
### (5). AbstractDelegatingConsumerEndpointParser
```
// <service-activator input-channel="inputChannel" output-channel="outputChannel" ref="helloService" method="sayHello"/>

protected final BeanDefinitionBuilder parseHandler(Element element, ParserContext parserContext) {
	Object source = parserContext.extractSource(element);
	// 定义Bean:ServiceActivatorFactoryBean
	BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition(this.getFactoryBeanClassName());
	// 解析子节点 <service-activator> <bean> ... </bean>  </service-activator>
	BeanComponentDefinition innerDefinition = IntegrationNamespaceUtils.parseInnerHandlerDefinition(element, parserContext);
	// ref
	String ref = element.getAttribute(REF_ATTRIBUTE);
	// expression
	String expression = element.getAttribute(EXPRESSION_ATTRIBUTE);
	
	boolean hasRef = StringUtils.hasText(ref);
	boolean hasExpression = StringUtils.hasText(expression);
	// 解析子节点  <service-activator> <script> ... </script>  </service-activator>
	Element scriptElement = DomUtils.getChildElementByTagName(element, "script");
	// 解析子节点  <service-activator> <expression> ... </expression>  </service-activator>
	Element expressionElement = DomUtils.getChildElementByTagName(element, "expression");
	
	// <bean>与ref/expression是排它的,否则,会抛出异常
	if (innerDefinition != null) {
		if (hasRef || hasExpression || expressionElement != null) {
			parserContext.getReaderContext().error("Neither 'ref' nor 'expression' are permitted when an inner bean (<bean/>) is configured on element " + IntegrationNamespaceUtils.createElementDescription(element) + ".", source);
			return null;
		}
		
		builder.addPropertyValue("targetObject", innerDefinition);
	} else if (scriptElement != null) {
		// <script>与ref/expression是排它的,否则,会抛出异常
		if (hasRef || hasExpression || expressionElement != null) {
			parserContext.getReaderContext().error("Neither 'ref' nor 'expression' are permitted when an inner script element is configured on element " + IntegrationNamespaceUtils.createElementDescription(element) + ".", source);
			return null;
		}
		
		BeanDefinition scriptBeanDefinition = parserContext.getDelegate().parseCustomElement(scriptElement, builder.getBeanDefinition());
		builder.addPropertyValue("targetObject", scriptBeanDefinition);
	} else if (expressionElement != null) {
		// <expression>与ref/expression是排它的,否则,会抛出异常
		if (hasRef || hasExpression) {
			parserContext.getReaderContext().error(
					"Neither 'ref' nor 'expression' are permitted when an inner 'expression' element is configured on element " + IntegrationNamespaceUtils.createElementDescription(element) + ".", source);
			return null;
		}
		
		BeanDefinitionBuilder dynamicExpressionBuilder = BeanDefinitionBuilder.genericBeanDefinition(
				DynamicExpression.class);
		String key = expressionElement.getAttribute("key");
		String expressionSourceReference = expressionElement.getAttribute("source");
		dynamicExpressionBuilder.addConstructorArgValue(key);
		dynamicExpressionBuilder.addConstructorArgReference(expressionSourceReference);
		builder.addPropertyValue("expression", dynamicExpressionBuilder.getBeanDefinition());
	} else if (hasRef && hasExpression) {
		parserContext.getReaderContext().error("Only one of 'ref' or 'expression' is permitted, not both, on element " + IntegrationNamespaceUtils.createElementDescription(element) + ".", source);
		return null;
	} else if (hasRef) {
		builder.addPropertyReference("targetObject", ref);
	} else if (hasExpression) {
		builder.addPropertyValue("expressionString", expression);
	} else if (!this.hasDefaultOption()) {
		parserContext.getReaderContext().error("Exactly one of the 'ref' attribute, 'expression' attribute, " +
				"or inner bean (<bean/>) definition is required for element " +
				IntegrationNamespaceUtils.createElementDescription(element) + ".", source);
		return null;
	}
	
	// 解析属性:method
	String method = element.getAttribute(METHOD_ATTRIBUTE);
	if (StringUtils.hasText(method)) {
		if (hasExpression || expressionElement != null) {
			parserContext.getReaderContext().error("A 'method' attribute is not permitted when configuring an 'expression' on element " + IntegrationNamespaceUtils.createElementDescription(element) + ".", source);
		}
		
		if (hasRef || innerDefinition != null) {
			builder.addPropertyValue("targetMethodName", method);
		} else {
			parserContext.getReaderContext().error("A 'method' attribute is only permitted when either " + "a 'ref' or inner-bean definition is provided on element " + IntegrationNamespaceUtils.createElementDescription(element) + ".", source);
		}
	} 
	
	IntegrationNamespaceUtils.setValueIfAttributeDefined(builder, element, "requires-reply");
	IntegrationNamespaceUtils.setValueIfAttributeDefined(builder, element, "send-timeout");
	this.postProcess(builder, element, parserContext);
	return builder;
}
```
### (6). ServiceActivatorParser
```
public class ServiceActivatorParser extends AbstractDelegatingConsumerEndpointParser {

	@Override
	String getFactoryBeanClassName() {
		return ServiceActivatorFactoryBean.class.getName();
	}

	@Override
	boolean hasDefaultOption() {
		return false;
	}

	@Override
	void postProcess(BeanDefinitionBuilder builder, Element element, ParserContext parserContext) {
		IntegrationNamespaceUtils.setValueIfAttributeDefined(builder, element, "async");
	}
}
```
### (7). 总结
ServiceActivatorParser的主要目的是把XML解析成业务模型,需要注意的是,它会向Spring容器中,注册两个Bean(ServiceActivatorFactoryBean和ConsumerEndpointFactoryBean).   
+ ServiceActivatorFactoryBean实际是:MessageHandler的实现类(它主要负责接收消息,并处理消息).      
+ ConsumerEndpointFactoryBean负责inputChannel与outputChannel的关联来着的.   