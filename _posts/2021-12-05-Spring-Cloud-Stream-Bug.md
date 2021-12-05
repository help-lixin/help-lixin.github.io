---
layout: post
title: 'Spring Cloud Stream Bug问题总结' 
date: 2021-12-05
author: 李新
tags:  SpringCloudStream 
---

### (1). 概述
进了新的公司,在整框架,结果,发现:Spring Cloud Stream和Validator整合在一起时,应用程序都无法启动,直接报错.  

### (2). 错误日志如下
> 从错误日志上能看出来,出现了循环依赖

```
Error starting ApplicationContext. To display the conditions report re-run your application with 'debug' enabled.
2021-12-05 11:37:03.154 ERROR 2174 --- [  restartedMain] o.s.b.d.LoggingFailureAnalysisReporter   : 

***************************
APPLICATION FAILED TO START
***************************

Description:

The dependencies of some of the beans in the application context form a cycle:

   org.springframework.rocketmq.spring.starter.internalRocketMQTransAnnotationProcessor defined in class path resource [com/alibaba/cloud/stream/binder/rocketmq/config/RocketMQComponent4BinderAutoConfiguration.class]
      ↓
   transactionHandlerRegistry defined in class path resource [com/alibaba/cloud/stream/binder/rocketmq/config/RocketMQComponent4BinderAutoConfiguration.class]
      ↓
   rocketMQTemplate defined in class path resource [com/alibaba/cloud/stream/binder/rocketmq/config/RocketMQComponent4BinderAutoConfiguration.class]
      ↓
   jacksonObjectMapper defined in class path resource [org/springframework/boot/autoconfigure/jackson/JacksonAutoConfiguration$JacksonObjectMapperConfiguration.class]
      ↓
   jacksonObjectMapperBuilder defined in class path resource [org/springframework/boot/autoconfigure/jackson/JacksonAutoConfiguration$JacksonObjectMapperBuilderConfiguration.class]
      ↓
   standardJacksonObjectMapperBuilderCustomizer defined in class path resource [org/springframework/boot/autoconfigure/jackson/JacksonAutoConfiguration$Jackson2ObjectMapperBuilderCustomizerConfiguration.class]
      ↓
   spring.jackson-org.springframework.boot.autoconfigure.jackson.JacksonProperties
┌─────┐
|  BindingHandlerAdvise defined in class path resource [org/springframework/cloud/stream/config/BindingServiceConfiguration.class]
↑     ↓
|  org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration$EnableWebMvcConfiguration
↑     ↓
|  spring.mvc-org.springframework.boot.autoconfigure.web.servlet.WebMvcProperties
└─────┘
```
### (3). 查看BindingServiceConfiguration定义内容
```
@Configuration
@EnableConfigurationProperties({ BindingServiceProperties.class, SpringIntegrationProperties.class, StreamFunctionProperties.class })
@Import({ DestinationPublishingMetricsAutoConfiguration.class, SpelExpressionConverterConfiguration.class })
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
@ConditionalOnBean(value = BinderTypeRegistry.class, search = SearchStrategy.CURRENT)
public class BindingServiceConfiguration {
	
	// ****************************************************************************************
	// 依赖:Validator
	// 细细看了下BindingHandlerAdvise好像没有地方使用,所以,我比较简单的做法是:直接把这个类从Spring容器中剔除.   
	// 或者,实现BindingHandlerAdvise,然后注入到Spring容器里.  
	// ****************************************************************************************
	@Bean
	public BindingHandlerAdvise BindingHandlerAdvise(@Nullable MappingsProvider[] providers, @Nullable Validator validator) {
		Map<ConfigurationPropertyName, ConfigurationPropertyName> additionalMappings = new HashMap<>();
		if (!ObjectUtils.isEmpty(providers)) {
			for (int i = 0; i < providers.length; i++) {
				MappingsProvider mappingsProvider = providers[i];
				additionalMappings.putAll(mappingsProvider.getDefaultMappings());
			}
		}
		return new BindingHandlerAdvise(additionalMappings, validator);
	} // end BindingHandlerAdvise
}	
```
### (4). BeanFactoryPostProcessor剔除Bean

```
@Configuration
public class HibernateValidatorConfiguration implements BeanFactoryPostProcessor {

    /**
     * 当HibernateValidator与Spring Cloud Stream相遇时,在启动时提示循环依赖,并报错,该方法,主要用于剔除Spring Cloud Stream对Validator的使用.
     *
     * @param configurableListableBeanFactory
     * @throws BeansException
     */
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory configurableListableBeanFactory) throws BeansException {
        if (configurableListableBeanFactory instanceof BeanDefinitionRegistry) {
            BeanDefinitionRegistry beanDefinitionRegistry = (BeanDefinitionRegistry) configurableListableBeanFactory;
            if (beanDefinitionRegistry.containsBeanDefinition("BindingHandlerAdvise")) {
                beanDefinitionRegistry.removeBeanDefinition("BindingHandlerAdvise");
            }
        }
    } // end postProcessBeanFactory
}	
```
### (5). 总结
没想到哈,Spring Cloud Stream与Spring Boot整合在一起时,还能出现这样的Bug.  