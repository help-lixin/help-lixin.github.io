---
layout: post
title: 'Apollo源码学习之:@EnableApolloConfig作用(五)'
date: 2021-07-16
author: 李新
tags:  Apollo
---

### (1). 概述
在这里,主要对@EnableApolloConfig注解,在底层它到底做了啥.  

### (2). EnableApolloConfig
```
// ***************************************************************
// 相当于spring里的<import file="xxx.xml"/>
// ***************************************************************
@Import(ApolloConfigRegistrar.class)
public @interface EnableApolloConfig {
  // 可以在注解上指定多个:namespace
  String[] value() default {ConfigConsts.NAMESPACE_APPLICATION};

	
  int order() default Ordered.LOWEST_PRECEDENCE;
}
```
### (3). ApolloConfigRegistrar
```
public class ApolloConfigRegistrar implements ImportBeanDefinitionRegistrar, EnvironmentAware {
  
  // 1. Apollo自定义的SPI加载器
  // Apollo这样写的原因是:它先写好了大部份功能,再向Spring靠扰而已
  private final ApolloConfigRegistrarHelper helper = ServiceBootstrap.loadPrimary(ApolloConfigRegistrarHelper.class);


  // *************************************************************************
  // 3. Spring初始化时,会回调:registerBeanDefinitions
  //    一看这个方法名称就知道,这个方法是向Spring中注册Bean.
  //    ApolloConfigRegistrarHelper的实现类是:DefaultApolloConfigRegistrarHelper
  // *************************************************************************
  @Override
  public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
    helper.registerBeanDefinitions(importingClassMetadata, registry);
  }

  @Override
  public void setEnvironment(Environment environment) {
	// 2. Spring初始化时,会回调:setEnvironment
    this.helper.setEnvironment(environment);
  }

}
```
### (4). DefaultApolloConfigRegistrarHelper
```
public class DefaultApolloConfigRegistrarHelper implements ApolloConfigRegistrarHelper {

  private Environment environment;

  @Override
  public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
	  // 获得App类上的注解:@EnableApolloConfig
    AnnotationAttributes attributes = AnnotationAttributes.fromMap(importingClassMetadata.getAnnotationAttributes(EnableApolloConfig.class.getName()));
	// 默认值就是: [application]
    final String[] namespaces = attributes.getStringArray("value");
	// 2147483647
    final int order = attributes.getNumber("order");
	final String[] resolvedNamespaces = this.resolveNamespaces(namespaces);
	// **************************************************************
	// 为PropertySourcesProcessor配置命名空间.
	// PropertySourcesProcessor最终是Properties的载体.
	// **************************************************************
    PropertySourcesProcessor.addNamespaces(Lists.newArrayList(resolvedNamespaces), order);

	// 
    Map<String, Object> propertySourcesPlaceholderPropertyValues = new HashMap<>();
    // to make sure the default PropertySourcesPlaceholderConfigurer's priority is higher than PropertyPlaceholderConfigurer
    propertySourcesPlaceholderPropertyValues.put("order", 0);

	// ***************************************************************
	// 向Spring中注册Bean
	// PropertySourcesPlaceholderConfigurer        : 这个是Spring自带的
	// PropertySourcesProcessor                    : 获得ConfigurableEnvironment对象,并创建PropertySource,并添加到:ConfigurableEnvironment的first.
	// ApolloAnnotationProcessor                   : 针对:@ApolloConfigChangeListener/@ApolloJsonValue等注解处理.
	// SpringValueProcessor                        : 针对@Value注解进行处理
	// SpringValueDefinitionProcessor              : 这个暂时不知道,是干嘛的
	// ***************************************************************
    BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, PropertySourcesPlaceholderConfigurer.class.getName(),PropertySourcesPlaceholderConfigurer.class, propertySourcesPlaceholderPropertyValues);
    BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, PropertySourcesProcessor.class.getName(),PropertySourcesProcessor.class);
    BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, ApolloAnnotationProcessor.class.getName(),ApolloAnnotationProcessor.class);
    BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, SpringValueProcessor.class.getName(),SpringValueProcessor.class);
    BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, SpringValueDefinitionProcessor.class.getName(),SpringValueDefinitionProcessor.class);
  }

  private String[] resolveNamespaces(String[] namespaces) {
    String[] resolvedNamespaces = new String[namespaces.length];
    for (int i = 0; i < namespaces.length; i++) {
      // throw IllegalArgumentException if given text is null or if any placeholders are unresolvable
      resolvedNamespaces[i] = this.environment.resolveRequiredPlaceholders(namespaces[i]);
    }
    return resolvedNamespaces;
  }

  @Override
  public int getOrder() {
    return Ordered.LOWEST_PRECEDENCE;
  }

  @Override
  public void setEnvironment(Environment environment) {
    this.environment = environment;
  }
}
```
### (5). 总结
@EnableApolloConfig的主要作用就是向Spring容器中注册一堆Bean,但是,好像也没有找到我们想要的内容.  
1) PropertySourcesPlaceholderConfigurer   
2) PropertySourcesProcessor   
3) ApolloAnnotationProcessor   
4) SpringValueProcessor   
5) SpringValueDefinitionProcessor
