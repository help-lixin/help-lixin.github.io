---
layout: post
title: 'Spring Cloud Stream源码(RocketMQ)'
date: 2020-07-03
author: 李新
tags: SpringCloudStream
---

### (1).叙述
> 在前面BinderFactoryConfiguration的binderTypeRegistry方法中可以看到.Spring解析了:META-INF/spring.binders.而该文件内容如下:
> spring-cloud-starter-stream-rocketmq-2.1.2.RELEASE.jar

```
rocketmq:com.alibaba.cloud.stream.binder.rocketmq.config.RocketMQBinderAutoConfiguration
```
### (2).RocketMQBinderHealthIndicatorAutoConfiguration
```
@Configuration
@ConditionalOnClass(Endpoint.class)
public class RocketMQBinderHealthIndicatorAutoConfiguration {

    // 1. 创建健康检查
	@Bean
	@ConditionalOnEnabledHealthIndicator("rocketmq")
	public RocketMQBinderHealthIndicator rocketBinderHealthIndicator() {
		return new RocketMQBinderHealthIndicator();
	}

}
```
### (3).RocketMQBinderAutoConfiguration
```
@Configuration
@Import({ RocketMQAutoConfiguration.class,
		RocketMQBinderHealthIndicatorAutoConfiguration.class })
@EnableConfigurationProperties({ RocketMQBinderConfigurationProperties.class,
		RocketMQExtendedBindingProperties.class })

public class RocketMQBinderAutoConfiguration {
    private final RocketMQExtendedBindingProperties extendedBindingProperties;

	private final RocketMQBinderConfigurationProperties rocketBinderConfigurationProperties;

	@Autowired(required = false)
	private RocketMQProperties rocketMQProperties = new RocketMQProperties();

	@Autowired
	public RocketMQBinderAutoConfiguration(
			RocketMQExtendedBindingProperties extendedBindingProperties,
			RocketMQBinderConfigurationProperties rocketBinderConfigurationProperties) {
		this.extendedBindingProperties = extendedBindingProperties;
		this.rocketBinderConfigurationProperties = rocketBinderConfigurationProperties;
	}
 
   // 2. 创建检测管理
   @Bean
	public InstrumentationManager instrumentationManager() {
		return new InstrumentationManager();
	}
 
   // 3. 创建 RocketMQTopicProvisioner
   @Bean
	public RocketMQTopicProvisioner provisioningProvider() {
		return new RocketMQTopicProvisioner();
	}
 
   // *************************************************************
   // 4. 创建RocketMessageChannelBinder
   //    RocketMQMessageChannelBinder实现了:org.springframework.cloud.stream.binder.Binder
   // *************************************************************
   @Bean
	public RocketMQMessageChannelBinder rocketMessageChannelBinder(
			RocketMQTopicProvisioner provisioningProvider,
			InstrumentationManager instrumentationManager) {
		RocketMQMessageChannelBinder binder = new RocketMQMessageChannelBinder(
				provisioningProvider, extendedBindingProperties,
				rocketBinderConfigurationProperties, rocketMQProperties,
				instrumentationManager);
		binder.setExtendedBindingProperties(extendedBindingProperties);
		return binder;
	}// end 

}
```
