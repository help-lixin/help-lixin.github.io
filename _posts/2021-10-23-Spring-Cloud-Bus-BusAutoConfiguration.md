---
layout: post
title: 'Spring Cloud Bus源码之BusAutoConfiguration(六)' 
date: 2021-10-23
author: 李新
tags:  SpringCloudBus
---

### (1). 概述
在前面,把BusAutoConfiguration初始化的一部份代码给剖析了一篇,在这一篇,主要剖析:BusAutoConfiguration.

### (2). BusAutoConfiguration
```
@Configuration
@ConditionalOnBusEnabled
@EnableBinding(SpringCloudBusClient.class)
@EnableConfigurationProperties(BusProperties.class)
@AutoConfigureBefore(BindingServiceConfiguration.class) // so stream bindings work properly
@AutoConfigureAfter(LifecycleMvcEndpointAutoConfiguration.class) // so actuator endpoints have needed dependencies
public class BusAutoConfiguration implements ApplicationEventPublisherAware {

	public static final String BUS_PATH_MATCHER_NAME = "busPathMatcher";

	public static final String CLOUD_CONFIG_NAME_PROPERTY = "spring.cloud.config.name";
	
	// *************************************************************************************
	// MessageChannel(消息发送)
	// *************************************************************************************
	private MessageChannel cloudBusOutboundChannel;

	private ApplicationEventPublisher applicationEventPublisher;

	private final ServiceMatcher serviceMatcher;

	private final BindingServiceProperties bindings;

	private final BusProperties bus;

	// *************************************************************************************
	// 1. 构建器注入:ServiceMatcher/BindingServiceProperties/BusProperties
	//   注意:BindingServiceProperties属于(stream): 
	//   org.springframework.cloud.stream.config.BindingServiceProperties
	// *************************************************************************************
	public BusAutoConfiguration(ServiceMatcher serviceMatcher, BindingServiceProperties bindings, BusProperties bus) {
		this.serviceMatcher = serviceMatcher;
		this.bindings = bindings;
		this.bus = bus;
	}

	// *************************************************************************************
	// 2. 额,spring cloud bus实际就是配置:spring cloud stream
	// *************************************************************************************
	@PostConstruct
	public void init() {
		
		// bindings是一个map集合(springCloudBusInput),从map中获取:BindingProperties
		// SpringCloudBusClient.INPUT = springCloudBusInput
		BindingProperties inputBinding = this.bindings.getBindings().get(SpringCloudBusClient.INPUT);
		// 如果不存在,则创建一个
		if (inputBinding == null) { // false
			this.bindings.getBindings().put(SpringCloudBusClient.INPUT, new BindingProperties());
		}
		
		BindingProperties input = this.bindings.getBindings().get(SpringCloudBusClient.INPUT);
		if (input.getDestination() == null || input.getDestination().equals(SpringCloudBusClient.INPUT)) {
			// *****************************************************************************************
			// 2.1 设置destination=springCloudBus
			// *****************************************************************************************
			input.setDestination(this.bus.getDestination());
		}
		
		// bindings是一个map集合(springCloudBusOutput),从map中获取:BindingProperties
		// SpringCloudBusClient.OUTPUT = springCloudBusOutput
		BindingProperties outputBinding = this.bindings.getBindings().get(SpringCloudBusClient.OUTPUT);
		if (outputBinding == null) {
			this.bindings.getBindings().put(SpringCloudBusClient.OUTPUT,new BindingProperties());
		}
		
		BindingProperties output = this.bindings.getBindings().get(SpringCloudBusClient.OUTPUT);
		if (output.getDestination() == null || output.getDestination().equals(SpringCloudBusClient.OUTPUT)) {
			// *****************************************************************************************
			// 2.2 设置destination=springCloudBus
			// *****************************************************************************************
			output.setDestination(this.bus.getDestination());
		}
	} // end init
	
	
	// *****************************************************************************************
	// 3. 注入MessageChannel
	// *****************************************************************************************
	@Autowired
	@Output(SpringCloudBusClient.OUTPUT)
	public void setCloudBusOutboundChannel(MessageChannel cloudBusOutboundChannel) {
		this.cloudBusOutboundChannel = cloudBusOutboundChannel;
	}// end setCloudBusOutboundChannel
}
```
### (3). 事件发布与监听
```
// 监听:RemoteApplicationEvent事件,往MessageChannel里发送数据
@EventListener(classes = RemoteApplicationEvent.class)
public void acceptLocal(RemoteApplicationEvent event) {
	if (this.serviceMatcher.isFromSelf(event)
			&& !(event instanceof AckRemoteApplicationEvent)) {
		this.cloudBusOutboundChannel.send(MessageBuilder.withPayload(event).build());
	}
}

// 订阅消息处理
@StreamListener(SpringCloudBusClient.INPUT)
public void acceptRemote(RemoteApplicationEvent event) {
	
	// 
	if (event instanceof AckRemoteApplicationEvent) {
		if (this.bus.getTrace().isEnabled() && !this.serviceMatcher.isFromSelf(event) && this.applicationEventPublisher != null) {
			this.applicationEventPublisher.publishEvent(event);
		}
		return;
	}
	
	if (this.serviceMatcher.isForSelf(event) && this.applicationEventPublisher != null) {
		if (!this.serviceMatcher.isFromSelf(event)) {
			this.applicationEventPublisher.publishEvent(event);
		}
		
		if (this.bus.getAck().isEnabled()) {
			AckRemoteApplicationEvent ack = new AckRemoteApplicationEvent(this,
					this.serviceMatcher.getServiceId(),
					this.bus.getAck().getDestinationService(),
					event.getDestinationService(), event.getId(), event.getClass());
             // 发送ack事件
			this.cloudBusOutboundChannel.send(MessageBuilder.withPayload(ack).build());
			this.applicationEventPublisher.publishEvent(ack);
		}
	}
	
	if (this.bus.getTrace().isEnabled() && this.applicationEventPublisher != null) {
		this.applicationEventPublisher.publishEvent(new SentApplicationEvent(this, event.getOriginService(), event.getDestinationService(),event.getId(), event.getClass()));
	}
}
```
### (4). 总结
Spring Cloud Bus的底层实际是组装配置(BindingServiceProperties),余下的事情交给了:Spring Cloud Stream做处理.   