---
layout: post
title: 'Spring Cloud Bus源码之RefreshBusEndpoint和EnvironmentBusEndpoint(五)' 
date: 2021-10-23
author: 李新
tags:  SpringCloudBus
---

### (1). 概述
前面已经把:spring.factories里内容分析差不多了,只有最后一个:BusAutoConfiguration类了,但是,由于这个类的内容比较多,所以,我这里,将会拆开来剖析,在这里:主要分析Spring Cloud Bus针对@Endpoint的扩展.     
### (2). BusAutoConfiguration
```
@Configuration
@ConditionalOnBusEnabled
// ************************************************************************************
// 1. spring cloud stream @EnableBinding
//    实际就是配置:MessageChannel/SubscribableChannel
// ************************************************************************************
@EnableBinding(SpringCloudBusClient.class)
@EnableConfigurationProperties(BusProperties.class)
@AutoConfigureBefore(BindingServiceConfiguration.class) // so stream bindings work properly
@AutoConfigureAfter(LifecycleMvcEndpointAutoConfiguration.class) // so actuator endpoints have needed dependencies

// ************************************************************************************
// 2. 实现了:ApplicationEventPublisherAware,在Spring初始化时,会回传:ApplicationEventPublisher对象
// ************************************************************************************
public class BusAutoConfiguration implements ApplicationEventPublisherAware {
   	
	
	@Configuration
	@ConditionalOnClass(EnvironmentManager.class)
	@ConditionalOnBean(EnvironmentManager.class)
	protected static class BusEnvironmentConfiguration {
		
		// ************************************************************************************
		// 4. 创建:EnvironmentChangeListener,当配置信息发生变化时的回调函数
		// ************************************************************************************
		@Bean
		@ConditionalOnProperty(value = "spring.cloud.bus.env.enabled", matchIfMissing = true)
		public EnvironmentChangeListener environmentChangeListener() {
			return new EnvironmentChangeListener();
		}
		
		// ************************************************************************************
		// 3. 创建EnvironmentBusEndpoint,这个类主要用于发布事件的(里面包裹着:ApplicationContext)
		//    http://domain/acturator/bus-env
		// ************************************************************************************
		@Configuration
		@ConditionalOnClass(Endpoint.class)
		protected static class EnvironmentBusEndpointConfiguration {
			
			@Bean
			@ConditionalOnEnabledEndpoint
			public EnvironmentBusEndpoint environmentBusEndpoint(ApplicationContext context, BusProperties bus) {
				// 这个内容是在:BusEnvironmentPostProcessor里配置的
				// bus.getId() = spring.cloud.bus.application
				// bus.getId() = spring-cloud-bus-server:7070:6013473fe09937d172b6de733db271c1
				return new EnvironmentBusEndpoint(context, bus.getId());
			}
		}
	} // end BusEnvironmentConfiguration
	
	
	@Configuration
	@ConditionalOnClass({ Endpoint.class, RefreshScope.class })
	protected static class BusRefreshConfiguration {

		@Configuration
		@ConditionalOnBean(ContextRefresher.class)
		protected static class BusRefreshEndpointConfiguration {
			
			// ************************************************************************************
			// 5. 定义spring-acturator的endpoint
			//    http://domain/acturator/bus-refresh
			// ************************************************************************************
			@Bean
			@ConditionalOnEnabledEndpoint
			public RefreshBusEndpoint refreshBusEndpoint(ApplicationContext context, BusProperties bus) {
				return new RefreshBusEndpoint(context, bus.getId());
			} // end refreshBusEndpoint
		} // end BusRefreshEndpointConfiguration
		
	} // end BusRefreshConfiguration
}	
```
### (3). EnvironmentBusEndpoint
```
package org.springframework.cloud.bus.endpoint;

// ************************************************************************************
// spring-acturator 定义@Endpoint
//  http://domain/acturator/bus-env
// ************************************************************************************
@Endpoint(id = "bus-env") //TODO: document
// ************************************************************************************
// 1. 继承于:AbstractBusEndpoint,AbstractBusEndpoint提供了publish(ApplicationEvent)
// ************************************************************************************
public class EnvironmentBusEndpoint extends AbstractBusEndpoint {

	public EnvironmentBusEndpoint(ApplicationEventPublisher context, String id) {
		super(context, id);
	}

	@WriteOperation
	public void busEnvWithDestination(String name, String value, @Selector String destination) { //TODO: document params
		Map<String, String> params = Collections.singletonMap(name, value);
		// 把数据经过:EnvironmentChangeRemoteApplicationEvent包装,并发布
		publish(new EnvironmentChangeRemoteApplicationEvent(this, getInstanceId(), destination, params));
	}

	@WriteOperation
	public void busEnv(String name, String value) { //TODO: document params
		Map<String, String> params = Collections.singletonMap(name, value);
		// 把数据经过:EnvironmentChangeRemoteApplicationEvent包装,并发布
		publish(new EnvironmentChangeRemoteApplicationEvent(this, getInstanceId(), null, params));
	}

}// end EnvironmentBusEndpoint
```
### (4). RefreshBusEndpoint
```
// ************************************************************************************
// spring-acturator 定义@Endpoint
//  http://domain/acturator/bus-refresh
// ************************************************************************************
@Endpoint(id = "bus-refresh") //TODO: document new id
// ************************************************************************************
// 1. 继承于:AbstractBusEndpoint,AbstractBusEndpoint提供了publish(ApplicationEvent)
// ************************************************************************************
public class RefreshBusEndpoint extends AbstractBusEndpoint {

	public RefreshBusEndpoint(ApplicationEventPublisher context, String id) {
		super(context, id);
	}

	@WriteOperation
	public void busRefreshWithDestination(@Selector String destination) { //TODO: document destination
		publish(new RefreshRemoteApplicationEvent(this, getInstanceId(), destination));
	}

	@WriteOperation
	public void busRefresh() {
		publish(new RefreshRemoteApplicationEvent(this, getInstanceId(), null));
	}
}
```
### (5). AbstractBusEndpoint
```
public class AbstractBusEndpoint {

	private ApplicationEventPublisher context;

	private String appId;

	public AbstractBusEndpoint(ApplicationEventPublisher context, String appId) {
		this.context = context;
		this.appId = appId;
	}

	protected String getInstanceId() {
		return this.appId;
	}

	// ************************************************************************************
	// 发布事件
	// ************************************************************************************
	protected void publish(ApplicationEvent event) {
		context.publishEvent(event);
	}

} // end AbstractBusEndpoint
```
### (6). 总结
BusAutoConfiguration内部的静态类(BusRefreshConfiguration/BusEnvironmentConfiguration),主要是实现了@Endpoint.   