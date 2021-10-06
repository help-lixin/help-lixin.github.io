---
layout: post
title: 'Spring Integration源码之IntegrationNamespaceHandler(三)' 
date: 2021-10-06
author: 李新
tags:  SpringIntegration
---

### (1). 概述

在这一小篇,主要剖析,当我们通过xml配置时,底层到底是如何实现的?可以配置哪些xml标签,以及标签有哪些属性?   

### (2). hello-world.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<!-- 1. 定义了xml的命名空间为:int和DTD -->
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:int="http://www.springframework.org/schema/integration"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/integration
        http://www.springframework.org/schema/integration/spring-integration.xsd">
	<!-- 2. 定义了网关 --> 	
    <int:gateway default-request-channel="inChannel" service-interface="help.lixin.integration.example.Application$Greeting"/>
    <int:channel id="inChannel"/>
    <int:service-activator input-channel="inChannel" method="sayHello">
        <bean class="help.lixin.integration.example.Application$HelloMessageProvider"/>
    </int:service-activator>
</beans>
```
### (3). 入口程序在哪?
我们通过配置XML,就可以让Spring Integeration实现编排的功能,那么入口程序在哪?   
其实和Spring Boot很类似(/META-INF/spring.factories),Spring XML命名空间的解析是在:/META-INF/spring.handlers里面定义的.   
### (4). spring-integration-core-4.3.5.RELEASE.jar/META-INF/spring.handlers
```
# 定义命名空间对应的解析类
http\://www.springframework.org/schema/integration=org.springframework.integration.config.xml.IntegrationNamespaceHandler
```
### (5). IntegrationNamespaceHandler
```
package org.springframework.integration.config.xml;

// ****************************************************************
// 1. 针对命名空间的解析,Spring要求实现:org.springframework.beans.factory.xml.NamespaceHandler
// ****************************************************************
public class IntegrationNamespaceHandler extends AbstractIntegrationNamespaceHandler {

	@Override
	public void init() {
		// ******************************************************************************
		// 2. 解析 <int:channel id="inChannel"/>  
		// ******************************************************************************
		registerBeanDefinitionParser("channel", new PointToPointChannelParser());
		registerBeanDefinitionParser("publish-subscribe-channel", new PublishSubscribeChannelParser());
		
		// ******************************************************************************
		// 3. 解析
		// <int:service-activator input-channel="inChannel" method="sayHello">
        //     <bean class="help.lixin.integration.example.Application$HelloMessageProvider"/>
		// </int:service-activator>
		// ******************************************************************************
		registerBeanDefinitionParser("service-activator", new ServiceActivatorParser());
		registerBeanDefinitionParser("transformer", new TransformerParser());
		registerBeanDefinitionParser("enricher", new EnricherParser());
		registerBeanDefinitionParser("filter", new FilterParser());
		registerBeanDefinitionParser("router", new DefaultRouterParser());
		registerBeanDefinitionParser("header-value-router", new HeaderValueRouterParser());
		registerBeanDefinitionParser("payload-type-router", new PayloadTypeRouterParser());
		registerBeanDefinitionParser("exception-type-router", new ErrorMessageExceptionTypeRouterParser());
		registerBeanDefinitionParser("recipient-list-router", new RecipientListRouterParser());
		registerBeanDefinitionParser("splitter", new SplitterParser());
		registerBeanDefinitionParser("aggregator", new AggregatorParser());
		registerBeanDefinitionParser("resequencer", new ResequencerParser());
		registerBeanDefinitionParser("header-enricher", new StandardHeaderEnricherParser());
		registerBeanDefinitionParser("header-filter", new HeaderFilterParser());
		registerBeanDefinitionParser("object-to-string-transformer", new ObjectToStringTransformerParser());
		registerBeanDefinitionParser("object-to-map-transformer", new ObjectToMapTransformerParser());
		registerBeanDefinitionParser("map-to-object-transformer", new MapToObjectTransformerParser());
		registerBeanDefinitionParser("object-to-json-transformer", new ObjectToJsonTransformerParser());
		registerBeanDefinitionParser("json-to-object-transformer", new JsonToObjectTransformerParser());
		registerBeanDefinitionParser("payload-serializing-transformer", new PayloadSerializingTransformerParser());
		registerBeanDefinitionParser("payload-deserializing-transformer", new PayloadDeserializingTransformerParser());
		registerBeanDefinitionParser("stream-transformer", new StreamTransformerParser());
		registerBeanDefinitionParser("claim-check-in", new ClaimCheckInParser());
		registerBeanDefinitionParser("syslog-to-map-transformer", new SyslogToMapTransformerParser());
		registerBeanDefinitionParser("claim-check-out", new ClaimCheckOutParser());
		registerBeanDefinitionParser("inbound-channel-adapter", new DefaultInboundChannelAdapterParser());
		registerBeanDefinitionParser("resource-inbound-channel-adapter", new ResourceInboundChannelAdapterParser());
		registerBeanDefinitionParser("outbound-channel-adapter", new DefaultOutboundChannelAdapterParser());
		registerBeanDefinitionParser("logging-channel-adapter", new LoggingChannelAdapterParser());
		
		// ******************************************************************************
		// 4. 解析:<int:gateway default-request-channel="inChannel" service-interface="help.lixin.integration.example.Application$Greeting"/>
		// ******************************************************************************
		registerBeanDefinitionParser("gateway", new GatewayParser());
		registerBeanDefinitionParser("delayer", new DelayerParser());
		registerBeanDefinitionParser("bridge", new BridgeParser());
		registerBeanDefinitionParser("chain", new ChainParser());
		registerBeanDefinitionParser("selector", new SelectorParser());
		registerBeanDefinitionParser("selector-chain", new SelectorChainParser());
		registerBeanDefinitionParser("poller", new PollerParser());
		registerBeanDefinitionParser("annotation-config", new AnnotationConfigParser());
		registerBeanDefinitionParser("application-event-multicaster", new ApplicationEventMulticasterParser());
		registerBeanDefinitionParser("publishing-interceptor", new PublishingInterceptorParser());
		registerBeanDefinitionParser("channel-interceptor", new GlobalChannelInterceptorParser());
		registerBeanDefinitionParser("converter", new ConverterParser());
		registerBeanDefinitionParser("message-history", new MessageHistoryParser());
		registerBeanDefinitionParser("control-bus", new ControlBusParser());
		registerBeanDefinitionParser("wire-tap", new GlobalWireTapParser());
		registerBeanDefinitionParser("transaction-synchronization-factory", new TransactionSynchronizationFactoryParser());
		registerBeanDefinitionParser("spel-function", new SpelFunctionParser());
		registerBeanDefinitionParser("spel-property-accessors", new SpelPropertyAccessorsParser());
		RetryAdviceParser retryParser = new RetryAdviceParser();
		registerBeanDefinitionParser("handler-retry-advice", retryParser);
		registerBeanDefinitionParser("retry-advice", retryParser);
		registerBeanDefinitionParser("scatter-gather", new ScatterGatherParser());
		registerBeanDefinitionParser("idempotent-receiver", new IdempotentReceiverInterceptorParser());
		registerBeanDefinitionParser("management", new IntegrationManagementParser());
		registerBeanDefinitionParser("barrier", new BarrierParser());
	}
}
```
### (6). 总结
在这里,我们先只找到XML相应的解析类,后面会对:PointToPointChannelParser/ServiceActivatorParser/GatewayParser进行源码剖析,以确切的知道每个XML标签有哪些配置项.   