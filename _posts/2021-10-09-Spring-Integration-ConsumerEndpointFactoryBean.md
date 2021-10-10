---
layout: post
title: 'Spring Integration源码之ConsumerEndpointFactoryBean(九)' 
date: 2021-10-09
author: 李新
tags:  SpringIntegration
---

### (1). 概述
在前面,有剖析过:ServiceActivatorParser会,向Spring中注册两个Bean(ConsumerEndpointFactoryBean/ServiceActivatorFactoryBean),在这一小篇,主要剖析:ConsumerEndpointFactoryBean.  

### (2). ConsumerEndpointFactoryBean
```
public class ConsumerEndpointFactoryBean
		//  ***************************************************************
		// 1. FactoryBean所以,需要在意:getObject
		//  ***************************************************************
		implements FactoryBean<AbstractEndpoint>, 
		           BeanFactoryAware, 
				   BeanNameAware, 
				   BeanClassLoaderAware,
				   //  ***************************************************************
				   // 2. InitializingBean所以,需要在意:afterPropertiesSet
                   //  ***************************************************************
				   InitializingBean, 
				   SmartLifecycle {
    
	@Override
	public AbstractEndpoint getObject() throws Exception {
		if (!this.initialized) {
			//  ***************************************************************
			// 3. 初始化,并且,确保只实例化一次
			//  ***************************************************************
			this.initializeEndpoint();
		}
		return this.endpoint;
	} // end getObject
	
	
	private void initializeEndpoint() throws Exception {
		synchronized (this.initializationMonitor) {
			if (this.initialized) {
				return;
			}
			
			
			MessageChannel channel = null;
			if (StringUtils.hasText(this.inputChannelName)) {
				// 根据channel名称(inputChannelName)从Spring容器中,获得:MessageChannel
				channel = this.channelResolver.resolveDestination(this.inputChannelName);
			}
			if (this.inputChannel != null) {
				channel = this.inputChannel;
			}
			
			Assert.state(channel != null, "one of inputChannelName or inputChannel is required");
			// 判断是否为:SubscribableChannel的实现类
			if (channel instanceof SubscribableChannel) {  // true
				Assert.isNull(this.pollerMetadata, "A poller should not be specified for endpoint '" + this.beanName + "', since '" + channel + "' is a SubscribableChannel (not pollable).");
				
				// *************************************************************************************
				// 创建:EventDrivenConsumer
				// 要注意:
				//    此处的:handler就是前面分析:ServiceActivator后产生的:ServiceActivatingHandler.  
				//    此处的:channel就是xml配置的inputChannel.
				// *************************************************************************************
				this.endpoint = new EventDrivenConsumer((SubscribableChannel) channel, this.handler);
				
				// 打印日志
				if (this.logger.isWarnEnabled() && Boolean.FALSE.equals(this.autoStartup) && channel instanceof FixedSubscriberChannel) {
					this.logger.warn("'autoStartup=\"false\"' has no effect when using a FixedSubscriberChannel");
				}
			} else if (channel instanceof PollableChannel) { // PollableChannel应该是poll的实现吧
				PollingConsumer pollingConsumer = new PollingConsumer((PollableChannel) channel, this.handler);
				if (this.pollerMetadata == null) {
					this.pollerMetadata = PollerMetadata.getDefaultPollerMetadata(this.beanFactory);
					Assert.notNull(this.pollerMetadata, "No poller has been defined for endpoint '" + this.beanName
							+ "', and no default poller is available within the context.");
				}
				pollingConsumer.setTaskExecutor(this.pollerMetadata.getTaskExecutor());
				pollingConsumer.setTrigger(this.pollerMetadata.getTrigger());
				pollingConsumer.setAdviceChain(this.pollerMetadata.getAdviceChain());
				pollingConsumer.setMaxMessagesPerPoll(this.pollerMetadata.getMaxMessagesPerPoll());

				pollingConsumer.setErrorHandler(this.pollerMetadata.getErrorHandler());

				pollingConsumer.setReceiveTimeout(this.pollerMetadata.getReceiveTimeout());
				pollingConsumer.setTransactionSynchronizationFactory(
						this.pollerMetadata.getTransactionSynchronizationFactory());
				pollingConsumer.setBeanClassLoader(this.beanClassLoader);
				pollingConsumer.setBeanFactory(this.beanFactory);
				this.endpoint = pollingConsumer;
			} else {
				throw new IllegalArgumentException("unsupported channel type: [" + channel.getClass() + "]");
			}
			
			this.endpoint.setBeanName(this.beanName);
			this.endpoint.setBeanFactory(this.beanFactory);
			if (this.autoStartup != null) {
				this.endpoint.setAutoStartup(this.autoStartup);
			}
			
			int phase = this.phase;
			if (!this.isPhaseSet && this.endpoint instanceof PollingConsumer) {
				phase = Integer.MAX_VALUE / 2;
			}
			
			this.endpoint.setPhase(phase);
			
			this.endpoint.afterPropertiesSet();
			this.initialized = true;
		}
	}// end initializeEndpoint

}
```
### (3). 总结
ConsumerEndpointFactoryBean底层,又委托给了:EventDrivenConsumer类,而:EventDrivenConsumer的构造器关联着inputChannel与MessageHandler(ServiceActivator).   