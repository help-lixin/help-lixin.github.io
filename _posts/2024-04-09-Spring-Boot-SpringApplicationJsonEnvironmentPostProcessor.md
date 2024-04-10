---
layout: post
title: 'Spring Boot之SpringApplicationJsonEnvironmentPostProcessor源码解析' 
date: 2024-03-16
author: 李新
tags:  SpringBoot
---

### (1). 背景

在Spring中,我一直所用的配置文件为(properties/yaml),但是,我这边需要做到动态配置,需要用到JSON,所以研究了下SpringBoot源码,发现底层是支持的.

### (2). SpringApplicationJsonEnvironmentPostProcessor
```
public class SpringApplicationJsonEnvironmentPostProcessor 
	// *******************************************************************
	// 1. 实现Env后置处理
	// *******************************************************************
	implements EnvironmentPostProcessor, Ordered {

	public static final String SPRING_APPLICATION_JSON_PROPERTY = "spring.application.json";
	
	public static final String SPRING_APPLICATION_JSON_ENVIRONMENT_VARIABLE = "SPRING_APPLICATION_JSON";
	
	@Override
	public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
		MutablePropertySources propertySources = environment.getPropertySources();
		propertySources.stream().map(JsonPropertyValue::get).filter(Objects::nonNull)
				// ******************************************************
				// 只取第一个
				// ******************************************************
				.findFirst()
				// ******************************************************
				// 2. 对JSON进行处理:processJson
				// ******************************************************
				.ifPresent((v) -> processJson(environment, v));
	} // end postProcessEnvironment
	
	// ***********************************************************************
	// 3. 通过JsonParser(Jackson)把JSON解析成Map
	// ***********************************************************************
	private void processJson(ConfigurableEnvironment environment, JsonPropertyValue propertyValue) {
		JsonParser parser = JsonParserFactory.getJsonParser();
		Map<String, Object> map = parser.parseMap(propertyValue.getJson());
		if (!map.isEmpty()) {
			// *******************************************************************
			// 4. 交给:flatten处理
			// *******************************************************************
			addJsonPropertySource(environment, new JsonPropertySource(propertyValue, flatten(map)));
		}
	} // end processJson
	
	
	private Map<String, Object> flatten(Map<String, Object> map) {
		Map<String, Object> result = new LinkedHashMap<>();
		flatten(null, result, map);
		return result;
	} // end flatten

	private void flatten(String prefix, Map<String, Object> result, Map<String, Object> map) {
		String namePrefix = (prefix != null) ? prefix + "." : "";
		map.forEach((key, value) -> extract(namePrefix + key, result, value));
	} // end flatten

	@SuppressWarnings("unchecked")
	private void extract(String name, Map<String, Object> result, Object value) {
		if (value instanceof Map) {
			if (CollectionUtils.isEmpty((Map<?, ?>) value)) {
				result.put(name, value);
				return;
			}
			flatten(name, result, (Map<String, Object>) value);
		}
		else if (value instanceof Collection) {
			if (CollectionUtils.isEmpty((Collection<?>) value)) {
				result.put(name, value);
				return;
			}
			int index = 0;
			for (Object object : (Collection<Object>) value) {
				extract(name + "[" + index + "]", result, object);
				index++;
			}
		}
		else {
			result.put(name, value);
		}
	} // end extract
}
```
### (3). 总结
Spring的源码,值得我们去学习的,永远会开出一堆的接口,允许我们自由切换,以及组合使用. 