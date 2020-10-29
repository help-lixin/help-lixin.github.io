---
layout: post
title: 'Spring Cloud Stream源码(BinderFactoryConfiguration)'
date: 2020-07-03
author: 李新
tags: SpringCloudStream
---

### (1).叙述
> 在前面说过,@EnableBinding内部使用了:    
    @Import({     
          BindingBeansRegistrar.class,     
          BinderFactoryConfiguration.class     
    }),这一小节主要讲解:BinderFactoryConfiguration

### (2).BinderFactoryConfiguration

```
package org.springframework.cloud.stream.config;

@Configuration
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
// 1.启用配置信息(BindingServiceProperties),读取配置文件信息并转换成模型.
@EnableConfigurationProperties({ BindingServiceProperties.class })
// 2.导入ContentTypeConfiguration
@Import({ContentTypeConfiguration.class})
public class BinderFactoryConfiguration {
    
    // **************************************************
    // 4.  读取:META-INF/spring.binders下的内容
    // rocketmq:com.alibaba.cloud.stream.binder.rocketmq.config.RocketMQBinderAutoConfiguration
    // **************************************************
    @Bean
	public BinderTypeRegistry binderTypeRegistry(ConfigurableApplicationContext configurableApplicationContext) {
        Map<String, BinderType> binderTypes = new HashMap<>();
        ClassLoader classLoader = configurableApplicationContext.getClassLoader();
        try {
            // **************************************************
            // 读取classLoader下:META-INF/spring.binders
            // spring-cloud-starter-stream-rocketmq-2.1.2.RELEASE.jar
            //    META-INF
            //            spring.binders
            // rocketmq:com.alibaba.cloud.stream.binder.rocketmq.config.RocketMQBinderAutoConfiguration
            // **************************************************
            Enumeration<URL> resources = classLoader.getResources("META-INF/spring.binders");
            if (!Boolean.valueOf(this.selfContained) && (resources == null || !resources.hasMoreElements())) {
                this.logger.debug("Failed to locate 'META-INF/spring.binders' resources on the classpath."
						+ " Assuming standard boot 'META-INF/spring.factories' configuration is used");
            } else {
                while (resources.hasMoreElements()) {
                        URL url = resources.nextElement();
                        UrlResource resource = new UrlResource(url);
                        for (BinderType binderType : parseBinderConfigurations(classLoader, resource)) {
                            // rocketmq
                            // com.alibaba.cloud.stream.binder.rocketmq.config.RocketMQBinderAutoConfiguration
                            binderTypes.put(binderType.getDefaultName(), binderType);
                        } //end for
                }// end while
            } //end else
        } catch (IOException | ClassNotFoundException e) {
			throw new BeanCreationException("Cannot create binder factory:", e);
        }
        // DefaultBinderTypeRegistry 为中介者模式
        return new DefaultBinderTypeRegistry(binderTypes);
	} // end binderTypeRegistry
 
 
    // 6. 创建MessageConverterConfigurer
    @Bean
    public MessageConverterConfigurer messageConverterConfigurer(
                             BindingServiceProperties bindingServiceProperties,
                             CompositeMessageConverterFactory compositeMessageConverterFactory) {
        return new MessageConverterConfigurer(bindingServiceProperties, compositeMessageConverterFactory);
	}// end messageConverterConfigurer
 
 
   // 7. 创建组合的消息通道配置 
   @Bean
	public CompositeMessageChannelConfigurer compositeMessageChannelConfigurer(
			MessageConverterConfigurer messageConverterConfigurer) {
        List<MessageChannelConfigurer> configurerList = new ArrayList<>();
        configurerList.add(messageConverterConfigurer);
        return new CompositeMessageChannelConfigurer(configurerList);
	}// end compositeMessageChannelConfigurer
    
    // *******************************************************
    // 8. 创建订阅通道工厂,属于:BindingTargetFactory的实现类
    // *******************************************************
    @Bean
	public SubscribableChannelBindingTargetFactory channelFactory(
			CompositeMessageChannelConfigurer compositeMessageChannelConfigurer) {
		return new SubscribableChannelBindingTargetFactory(compositeMessageChannelConfigurer);
	}// end channelFactory
 
 
   // *******************************************************
   // 9. 创建Source通道工厂,属于:BindingTargetFactory的实现类
   // *******************************************************
   @Bean
	public MessageSourceBindingTargetFactory messageSourceFactory(CompositeMessageConverterFactory compositeMessageConverterFactory,
								CompositeMessageChannelConfigurer compositeMessageChannelConfigurer) {
		return new MessageSourceBindingTargetFactory(compositeMessageConverterFactory.getMessageConverterForAllRegistered(), compositeMessageChannelConfigurer);
	}// end messageSourceFactory
    
}
```
### (3).BindingServiceProperties
```
// 3.指定配置信息的前缀
@ConfigurationProperties("spring.cloud.stream")
public class BindingServiceProperties 
       implements ApplicationContextAware, InitializingBean {
    
    private ConfigurableApplicationContext applicationContext = new GenericApplicationContext();
    private ConversionService conversionService;
    
     // 配置文件中的信息
    // spring.cloud.stream.bindings.output.destination=test-topic
    private Map<String, BindingProperties> bindings = new TreeMap<>(String.CASE_INSENSITIVE_ORDER);
    // bindings =  
    // {
    //   output:{  
    //             group = null , 
    //             binder=null , 
    //             consumer={} , 
    //             producer={} , 
    //             destination:"test-topic" , 
    //             contentType:"application/json"  
    //   } //end output 
    // }
    
    // 3.1 Spring回调
    public void setApplicationContext(ApplicationContext applicationContext)
			throws BeansException {
		this.applicationContext = (ConfigurableApplicationContext) applicationContext;
            // 根据名字(integrationConversionService),从Spring容器上获得:GenericConversionService
		GenericConversionService cs = (GenericConversionService) IntegrationUtils.getConversionService(this.applicationContext.getBeanFactory());
		if (this.applicationContext.containsBean("spelConverter")) {
                // 添加指定的bean:spelConverter  
                Converter<?,?> converter = (Converter<?, ?>) this.applicationContext.getBean("spelConverter");
                cs.addConverter(converter);
		}
	}// end setApplicationContext
    
    // 3.2.Spring初始化bean后:回调
     public void afterPropertiesSet() throws Exception {
            // 从Spring容器上下文中获得:ConversionService
		if (this.conversionService == null) {
			this.conversionService = this.applicationContext.getBean(
					IntegrationUtils.INTEGRATION_CONVERSION_SERVICE_BEAN_NAME,
					ConversionService.class);
		}
	} //end afterPropertiesSet
    
}
```
### (4).ContentTypeConfiguration
```
@Configuration
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
class ContentTypeConfiguration {
    
    // 5. 创建一个组合式的消息转换工厂
    // converters = [
    // org.springframework.cloud.stream.converter.ApplicationJsonMessageMarshallingConverter
    // org.springframework.cloud.stream.converter.TupleJsonMessageConverter
    // org.springframework.cloud.stream.converter.CompositeMessageConverterFactory$1
    // org.springframework.cloud.stream.converter.ObjectStringMessageConverter
   // org.springframework.cloud.stream.converter.JavaSerializationMessageConverter
   // org.springframework.cloud.stream.converter.KryoMessageConverter
   // org.springframework.cloud.stream.converter.JsonUnmarshallingConverter
   // ]
    @Bean
	public CompositeMessageConverterFactory compositeMessageConverterFactory(
                    //org.springframework.beans.factory.support.DefaultListableBeanFactory$DependencyObjectProvider
			ObjectProvider<ObjectMapper> objectMapperObjectProvider,
                   // []
			@StreamMessageConverter List<MessageConverter> customMessageConverters) {
        return new CompositeMessageConverterFactory(customMessageConverters, objectMapperObjectProvider.getIfAvailable(ObjectMapper::new));
	}// end CompositeMessageConverterFactory
 
    
}
```
### (5).总结
> 1. 将配置文件中的信息,转化为模型:BindingServiceProperties    
> 2. 读取META-INF/spring.binders文件,并转化为:BinderTypeRegistry
> 3. 创建组合式的消息转换工厂(CompositeMessageConverterFactory)   
> 4. 创建MessageConverterConfigurer,包裹:    
        BindingServiceProperties    
        CompositeMessageConverterFactory    
> 5. 创建CompositeMessageChannelConfigurer,包裹:MessageConverterConfigurer
> 6. 创建SubscribableChannelBindingTargetFactory,包裹:CompositeMessageChannelConfigurer
> 7. 创建MessageSourceBindingTargetFactory,包裹:    
    CompositeMessageConverterFactory    
    CompositeMessageChannelConfigurer