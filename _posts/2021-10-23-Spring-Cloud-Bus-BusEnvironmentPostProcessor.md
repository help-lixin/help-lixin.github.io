---
layout: post
title: 'Spring Cloud Bus源码之BusEnvironmentPostProcessor(三)' 
date: 2021-10-23
author: 李新
tags:  SpringCloudBus
---

### (1). 概述
前面,通过一个小小的案例,有了一个入门,在这里,开始深入研究下:Spring Cloud Bus的源码,入口点在于:spring.factories 

### (2). BusEnvironmentPostProcessor
> BusEnvironmentPostProcessor实现了EnvironmentPostProcessor,所以,Spring启动时,会首先回调:EnvironmentPostProcessor的所有实现.通过EnvironmentPostProcessor我们可以扩展自定义的配置信息.  

```
package org.springframework.cloud.bus;

import java.util.HashMap;
import java.util.Map;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.env.EnvironmentPostProcessor;
import org.springframework.core.env.ConfigurableEnvironment;
import org.springframework.core.env.MapPropertySource;
import org.springframework.core.env.MutablePropertySources;
import org.springframework.core.env.PropertySource;


public class BusEnvironmentPostProcessor implements EnvironmentPostProcessor {

	private static final String PROPERTY_SOURCE_NAME = "defaultProperties";

	@Override
	public void postProcessEnvironment(ConfigurableEnvironment environment,
			SpringApplication application) {
		Map<String, Object> map = new HashMap<String, Object>();
		
		// 1. 构建参数
		// key    = spring.cloud.stream.bindings.springCloudBusOutput.content-type
		// value  = application/json 
		map.put("spring.cloud.stream.bindings." + SpringCloudBusClient.OUTPUT + ".content-type" , environment.getProperty("spring.cloud.bus.content-type", "application/json"));
		
		// key   = spring.cloud.bus.id
		// value = ${vcap.application.name:${spring.application.name:application}}:${vcap.application.instance_index:${spring.application.index:${local.server.port:${server.port:0}}}}:${vcap.application.instance_id:${random.value}} 
		// value的实际值为:${spring.application.name}:${server.port}:${random}
		map.put("spring.cloud.bus.id", getDefaultServiceId(environment));
		
		// 2. 往:Env(MutablePropertySources)里添加:PropertySource
		addOrReplace(environment.getPropertySources(), map);
	} // end postProcessEnvironment

	private void addOrReplace(MutablePropertySources propertySources, Map<String, Object> map) {
		MapPropertySource target = null;
		
		// 3. 判断:defaultProperties在env中是否存在
		if (propertySources.contains(PROPERTY_SOURCE_NAME)) {
			// 3.1 拿出对应的:PropertySource
			PropertySource<?> source = propertySources.get(PROPERTY_SOURCE_NAME);
			if (source instanceof MapPropertySource) {
				target = (MapPropertySource) source;
				// 3.2 遍历map,验证key是否存在,如果不存在,则添加.
				for (String key : map.keySet()) {
					if (!target.containsProperty(key)) {
						target.getSource().put(key, map.get(key));
					}
				}
			}
		}
		
		// 4. 构建一个:PropertySource
		if (target == null) {
			target = new MapPropertySource(PROPERTY_SOURCE_NAME, map);
		}
		
		// 5. 添加到MutablePropertySources的末尾.
		if (!propertySources.contains(PROPERTY_SOURCE_NAME)) {
			propertySources.addLast(target);
		}
	}// end addOrReplace
	
	
	// TODO: move this to commons
	private String getDefaultServiceId(ConfigurableEnvironment environment) {
		return "${vcap.application.name:${spring.application.name:application}}:${vcap.application.instance_index:${spring.application.index:${local.server.port:${server.port:0}}}}:${vcap.application.instance_id:${random.value}}";
	} // end getDefaultServiceId
}
```
### (3). 总结
BusEnvironmentPostProcessor好像比较简单,就是在Env(MutablePropertySources),里添加一个:PropertySource(PropertySource的配置信息的载体).   