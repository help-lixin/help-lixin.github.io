---
layout: post
title: 'Spring Integration源码之ServiceActivatorParser(五)' 
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
### (3). ServiceActivatorParser
```
// ServiceActivatorParser相当的简单,所有业务逻辑都在:AbstractDelegatingConsumerEndpointParser里,从这个抽象类的名字来看,是跟消费者有关系.
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
### (4). AbstractDelegatingConsumerEndpointParser
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
### (5). 总结
ServiceActivatorParser的主要目的是把XML解析成业务模型:ServiceActivatorFactoryBean,注意:这个类是FactoryBean的子类来着的,需要重点关注getObject/getObjectType来着的,下一小篇,主要剖析:ServiceActivatorFactoryBean.     