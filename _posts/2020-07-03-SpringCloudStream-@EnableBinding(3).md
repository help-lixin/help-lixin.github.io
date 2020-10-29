---
layout: post
title: 'Spring Cloud Stream源码(ContentTypeConfiguration)'
date: 2020-07-03
author: 李新
tags: SpringCloudStream
---

### (1).ContentTypeConfiguration
```
@Configuration
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
class ContentTypeConfiguration {
    
    // 1.创建ConfigurableCompositeMessageConverter
    @Bean(name = IntegrationContextUtils.ARGUMENT_RESOLVER_MESSAGE_CONVERTER_BEAN_NAME)
	public ConfigurableCompositeMessageConverter configurableCompositeMessageConverter(
			CompositeMessageConverterFactory factory) {

		return new ConfigurableCompositeMessageConverter(factory.getMessageConverterForAllRegistered().getConverters());
	}//end configurableCompositeMessageConverter
 
   
}
```
### (2).BinderFactoryConfiguration
```
@Configuration
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
@EnableConfigurationProperties({ BindingServiceProperties.class })
@Import({ContentTypeConfiguration.class})
@Deprecated
public class BinderFactoryConfiguration {

    // 2. 消息处理工厂    
    @Bean
	public static MessageHandlerMethodFactory messageHandlerMethodFactory(
            // org.springframework.cloud.stream.converter.CompositeMessageConverterFactory
            CompositeMessageConverterFactory compositeMessageConverterFactory,
            // org.springframework.integration.handler.support.HandlerMethodArgumentResolversHolder
            @Qualifier(IntegrationContextUtils.ARGUMENT_RESOLVERS_BEAN_NAME) HandlerMethodArgumentResolversHolder ahmar,
            // org.springframework.validation.beanvalidation.LocalValidatorFactoryBean
            @Nullable Validator validator,  
            // org.springframework.beans.factory.support.DefaultListableBeanFactory
            ConfigurableListableBeanFactory clbf) {
    	
		DefaultMessageHandlerMethodFactory messageHandlerMethodFactory = new DefaultMessageHandlerMethodFactory();
		messageHandlerMethodFactory.setMessageConverter(compositeMessageConverterFactory.getMessageConverterForAllRegistered());
		
		/*
		 * We essentially do the same thing as the DefaultMessageHandlerMethodFactory.initArgumentResolvers(..).
		 * We can't do it as custom resolvers for two reasons. 
		 * 1. We would have two duplicate (compatible) resolvers, so they would need to be ordered properly 
		 *    to ensure these new resolvers take precedence.
		 * 2. DefaultMessageHandlerMethodFactory.initArgumentResolvers(..) puts MessageMethodArgumentResolver 
		 * before custom converters thus not allowing an override which kind of proves #1.
		 * 
		 * In all, all this will be obsolete once https://jira.spring.io/browse/SPR-17503 is addressed and we can fall back on 
		 * core resolvers
		 */
		List<HandlerMethodArgumentResolver> resolvers = new LinkedList<>();
		resolvers.add(new SmartPayloadArgumentResolver(compositeMessageConverterFactory.getMessageConverterForAllRegistered(), validator));
		resolvers.add(new SmartMessageMethodArgumentResolver(compositeMessageConverterFactory.getMessageConverterForAllRegistered()));
		resolvers.add(new HeaderMethodArgumentResolver(null, clbf));
		resolvers.add(new HeadersMethodArgumentResolver());
		resolvers.addAll(ahmar.getResolvers());
		
		// modify HandlerMethodArgumentResolversHolder
		Field field = ReflectionUtils.findField(HandlerMethodArgumentResolversHolder.class, "resolvers");
		field.setAccessible(true);
		((List<?>) ReflectionUtils.getField(field, ahmar)).clear();	
		resolvers.forEach(ahmar::addResolver);
		// --
		
		messageHandlerMethodFactory.setArgumentResolvers(resolvers);
		messageHandlerMethodFactory.setValidator(validator);
		return messageHandlerMethodFactory;
	}// end messageHandlerMethodFactory
    
}
```

