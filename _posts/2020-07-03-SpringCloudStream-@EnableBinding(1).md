---
layout: post
title: 'Spring Cloud Stream源码(EnableBinding)'
date: 2020-07-03
author: 李新
tags: SpringCloudStream
---

### (1).注解入口
```
@EnableBinding({ Source.class })
```

```
## @EnableBinding 源码内容
// ...
// 导入Bean
// ****************************************************************
// 1.找到程序入口
// ****************************************************************
// 所以入口在:BindingBeansRegistrar和BinderFactoryConfiguration中
@Import({ BindingBeansRegistrar.class, BinderFactoryConfiguration.class})

// 启用Integration
// 参考Spring Integration文档
@EnableIntegration
public @interface EnableBinding {
	Class<?>[] value() default {};
}
```

```
package org.springframework.cloud.stream.messaging;

import org.springframework.cloud.stream.annotation.Output;
import org.springframework.messaging.MessageChannel;


public interface Source {
	String OUTPUT = "output";
 
	@Output(Source.OUTPUT)
	MessageChannel output();

}
```
### (2).BindingBeansRegistrar
```
public class BindingBeansRegistrar implements ImportBeanDefinitionRegistrar {

	@Override
	public void registerBeanDefinitions(AnnotationMetadata metadata,
			BeanDefinitionRegistry registry) {
        // 2. 获取@EnableBinding({ Source.class })注解上的属性
        AnnotationAttributes attrs = AnnotatedElementUtils.getMergedAnnotationAttributes( ClassUtils.resolveClassName(metadata.getClassName(), null), EnableBinding.class);
        for (Class<?> type : collectClasses(attrs, metadata.getClassName())) {
             // 判断Spring容器中:org.springframework.cloud.stream.messaging.Source
            //  是否存在
            if (!registry.containsBeanDefinition(type.getName())) { // (!false)
                // ***********************************************************
                // 4.向Spring容器注册bean
                // 4.1 获取@EnableBinding({ Source.class }) 上的Source接口
                // 4.2 解析Source上的注解@Output("output")
                // 4.3 向Spring中注册Bean信息(name=output,value=org.springframework.cloud.stream.messaging.Source) 
                // ***********************************************************
                // 委托给:BindingBeanDefinitionRegistryUtils进行bean注册
                BindingBeanDefinitionRegistryUtils.registerBindingTargetBeanDefinitions(type, type.getName(), registry);
                // 
                BindingBeanDefinitionRegistryUtils.registerBindingTargetsQualifiedBeanDefinitions(ClassUtils.resolveClassName(metadata.getClassName(), null), type, registry);
            } //end if
        } //end for
	}

   // 3. 获取@EnableBinding上的value属性
	private Class<?>[] collectClasses(AnnotationAttributes attrs, String className) {
		EnableBinding enableBinding = AnnotationUtils.synthesizeAnnotation(attrs,
				EnableBinding.class, ClassUtils.resolveClassName(className, null));
		return enableBinding.value();
	} //end collectClasses

}
```
### (3).BindingBeanDefinitionRegistryUtils
```
public abstract class BindingBeanDefinitionRegistryUtils {
    
    // ******************************************************************
    // 4. 注册Binding
    // ******************************************************************
    public static void registerBindingTargetBeanDefinitions(
                       // org.springframework.cloud.stream.messaging.Source
                       Class<?> type, 
                       // org.springframework.cloud.stream.messaging.Source
                       final String bindingTargetInterfaceBeanName,
                      // bean注册器
			   final BeanDefinitionRegistry registry) {
        ReflectionUtils.doWithMethods(type, method -> {
            // 遍历:org.springframework.cloud.stream.messaging.Source
            // 类上所有的方法,判断是否存在:@Input注解
            Input input = AnnotationUtils.findAnnotation(method, Input.class);
            if (input != null) { 
                String name = getBindingTargetName(input, method);
                registerInputBindingTargetBeanDefinition(input.value(), name, bindingTargetInterfaceBeanName,
        		method.getName(), registry);
            } //end if

            // ***************************************************************
            // 遍历:org.springframework.cloud.stream.messaging.Source
            // 类上所有的方法,判断是否存在:@Output注解
            // @Output(value="output")
            // ***************************************************************
            Output output = AnnotationUtils.findAnnotation(method, Output.class);
            if (output != null) {
                // 获得注解上的value
                // output
                String name = getBindingTargetName(output, method);
                // 注解OutputBindingTargetBeanDefinition
                // ***************************************************
                // 5.开始向Spring容器中进行Bean注册
                // ***************************************************
                registerOutputBindingTargetBeanDefinition(output.value(), name, bindingTargetInterfaceBeanName,
        		method.getName(), registry);
            }// end if
        });
	}// end registerBindingTargetBeanDefinitions

    public static void registerOutputBindingTargetBeanDefinition(
             // output
             String qualifierValue, 
             // output
             String name,
             // org.springframework.cloud.stream.messaging.Source
             String bindingTargetInterfaceBeanName, 
             // output
             String bindingTargetInterfaceMethodName,
             // bean注册器
             BeanDefinitionRegistry registry) {
        // *************************************************************
        // 委托给:registerBindingTargetBeanDefinition
        // *************************************************************
        registerBindingTargetBeanDefinition(Output.class, qualifierValue, name, bindingTargetInterfaceBeanName, bindingTargetInterfaceMethodName, registry);
	}// end registerOutputBindingTargetBeanDefinition
 
 
     private static void registerBindingTargetBeanDefinition(
                    // @Output.class
                    Class<? extends Annotation> qualifier,
                    // output
			String qualifierValue, 
                    // output
                    String name, 
                    // org.springframework.cloud.stream.messaging.Source
                    String bindingTargetInterfaceBeanName,
                    // output
			String bindingTargetInterfaceMethodName, 
                   // bean注册器
                    BeanDefinitionRegistry registry) {
           // 判断bean是否存在(output)
           // 看这样纸,是会创建一个bean.对应的@Output(value="output")
		if (registry.containsBeanDefinition(name)) { // false
			throw new BeanDefinitionStoreException(bindingTargetInterfaceBeanName, name,
					"bean definition with this name already exists - " + registry.getBeanDefinition(name));
		}
  
            // 创建Bean定义信息
		RootBeanDefinition rootBeanDefinition = new RootBeanDefinition();
           // 指定使用工厂bean的名称
           // org.springframework.cloud.stream.messaging.Source
		rootBeanDefinition.setFactoryBeanName(bindingTargetInterfaceBeanName);
           // 方法名称
           // output
		rootBeanDefinition.setUniqueFactoryMethodName(bindingTargetInterfaceMethodName);
           // 注册自动注入@Auto....
           // qualifier = org.springframework.cloud.stream.annotation.Output
           // qualifier = output
		rootBeanDefinition.addQualifier(new AutowireCandidateQualifier(qualifier, qualifierValue));
           // name = output
		registry.registerBeanDefinition(name, rootBeanDefinition);
	}// end registerBindingTargetBeanDefinition


   // 获得注解上的value属性,如果没有配置value属性,返回的是:方法名称
    public static String getBindingTargetName(Annotation annotation, Method method) {
        Map<String, Object> attrs = AnnotationUtils.getAnnotationAttributes(annotation, false);
        if (attrs.containsKey("value") && StringUtils.hasText((CharSequence) attrs.get("value"))) {
            return (String) attrs.get("value");
        }
        return method.getName();
	}// end getBindingTargetName
}
```
### (4).BindingBeansRegistrar
```
public class BindingBeansRegistrar implements ImportBeanDefinitionRegistrar {
    public static void registerBindingTargetsQualifiedBeanDefinitions(
                 // help.lixin.samples.StreamSourceApplication
                 Class<?> parent, 
                 // org.springframework.cloud.stream.messaging.Source
                 Class<?> type,
                 // bean注册器
                 final BeanDefinitionRegistry registry) {

        // type = org.springframework.cloud.stream.messaging.Source
        if (type.isInterface()) { // true
            // ****************************************************************
            // 创建Bean定义(BindableProxyFactory)
            // ****************************************************************
            RootBeanDefinition rootBeanDefinition = new RootBeanDefinition(BindableProxyFactory.class);
            // 注册依赖置入功能@Auto...
            // 
            rootBeanDefinition.addQualifier(new AutowireCandidateQualifier(Bindings.class, parent));
            // 添加构造器:org.springframework.cloud.stream.messaging.Source
            rootBeanDefinition.getConstructorArgumentValues().addGenericArgumentValue(type);
            // type.name = org.springframework.cloud.stream.messaging.Source
            registry.registerBeanDefinition(type.getName(), rootBeanDefinition);
        } else {
            RootBeanDefinition rootBeanDefinition = new RootBeanDefinition(type);
            rootBeanDefinition.addQualifier(new AutowireCandidateQualifier(Bindings.class, parent));
            registry.registerBeanDefinition(type.getName(), rootBeanDefinition);
        } //end else
	}// end registerBindingTargetsQualifiedBeanDefinitions
}
```
### (5).总结
> 1. 启用@EnableBinding({ Source.class })
> 2. BindingBeansRegistrar会解析该注解配置的value:Source.class
> 3. Source是一个接口,并配置有注解:@Output(value="output")
> 3. Spring会向容器中注册bean.    
		beanName:output    
		value:?????
> 4. Spring会向容器中注册bean.  
		beanName:org.springframework.cloud.stream.messaging.Source
		value:BindableProxyFactory