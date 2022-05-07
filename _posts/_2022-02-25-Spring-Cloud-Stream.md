---
layout: post
title: 'Spring Cloud Stream源码之BinderFactoryAutoConfiguration(一)' 
date: 2022-02-25
author: 李新
tags:  SpringCloudStream
---

### (1). 概述


### (2). BinderFactoryAutoConfiguration
```
@Configuration
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
@EnableConfigurationProperties({ BindingServiceProperties.class })
@Import(ContentTypeConfiguration.class)
public class BinderFactoryAutoConfiguration {
   
	@Bean
	public BinderTypeRegistry binderTypeRegistry(ConfigurableApplicationContext configurableApplicationContext) {
		Map<String, BinderType> binderTypes = new HashMap<>();
		ClassLoader classLoader = configurableApplicationContext.getClassLoader();
		try {
			// ****************************************************************************
			// 1. 读取:META-INF/spring.binders文件.
			// ****************************************************************************
			Enumeration<URL> resources = classLoader.getResources("META-INF/spring.binders");
			
			// see if test binder is available on the classpath and if so add it to the binderTypes
			try {
				
				// 2. 尝试加载:org.springframework.cloud.stream.binder.test.TestChannelBinderConfiguration
				BinderType bt = new BinderType("integration", new Class[] {classLoader.loadClass("org.springframework.cloud.stream.binder.test.TestChannelBinderConfiguration")});
				binderTypes.put("integration", bt);
			} catch (Exception e) {
				// ignore. means test binder is not available
			}
			
			// 3. 如果没有读取到的情况下,打印错误信息
			if (binderTypes.isEmpty() && !Boolean.valueOf(this.selfContained) && (resources == null || !resources.hasMoreElements())) {
				this.logger.debug("Failed to locate 'META-INF/spring.binders' resources on the classpath."
								+ " Assuming standard boot 'META-INF/spring.factories' configuration is used");
			} else {
				//4. 遍历
				while (resources.hasMoreElements()) {
					URL url = resources.nextElement();
					UrlResource resource = new UrlResource(url);
					// *************************************************************************
					// 5. 解析:spring.binders里的内容
					// *************************************************************************
					for (BinderType binderType : parseBinderConfigurations(classLoader, resource)) {
						binderTypes.put(binderType.getDefaultName(), binderType);
					}
				}
			}
		} catch (IOException | ClassNotFoundException e) { 
			throw new BeanCreationException("Cannot create binder factory:", e);
		}
		// 6. 通过DefaultBinderTypeRegistry保存所有的:BinderType信息.
		return new DefaultBinderTypeRegistry(binderTypes);
	}  // end binderTypeRegistry
	
	
	static Collection<BinderType> parseBinderConfigurations(ClassLoader classLoader, Resource resource) throws IOException, ClassNotFoundException {
		Properties properties = PropertiesLoaderUtils.loadProperties(resource);
		Collection<BinderType> parsedBinderConfigurations = new ArrayList<>();
		for (Map.Entry<?, ?> entry : properties.entrySet()) {
			// 5.1 rocketmq
			String binderType = (String) entry.getKey();
			// 5.2 com.alibaba.cloud.stream.binder.rocketmq.config.RocketMQBinderAutoConfiguration
			String[] binderConfigurationClassNames = StringUtils.commaDelimitedListToStringArray((String) entry.getValue());
			
			Class<?>[] binderConfigurationClasses = new Class[binderConfigurationClassNames.length];
			int i = 0;
			for (String binderConfigurationClassName : binderConfigurationClassNames) {
				// 5.3 通过反射加载类
				binderConfigurationClasses[i++] = ClassUtils.forName(binderConfigurationClassName, classLoader);
			}
			parsedBinderConfigurations.add(new BinderType(binderType, binderConfigurationClasses));
		}
		return parsedBinderConfigurations;
	} // end parseBinderConfigurations
	
}	
```
### (3). 
```

FunctionProperties(spring.cloud.function)


ContextFunctionCatalogAutoConfiguration.functionCatalog
FunctionConfiguration.streamBridgeUtils
ContextFunctionCatalogAutoConfiguration.functionRouter

FunctionConfiguration.functionBindingRegistrar
FunctionConfiguration.functionInitializer
FunctionConfiguration.supplierInitializer



BindableFunctionProxyFactory
FunctionToDestinationBinder

```
### (4). 

### (5). 

### (6). 

### (7). 

### (8). 

### (9). 

### (10). 
 