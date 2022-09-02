---
layout: post
title: 'Spring Plugin简单使用' 
date: 2022-09-02
author: 李新
tags:  SpringPlugin
---

### (1). 概述
近来的事情实在的太多了,很少更新内容,偶尔上Github看一下有没有什么值得学习的技术,无意间看到Spring Shell和Spring Plugin,而对于Plugin一直情有独钟,期望Spring生态圈中找到我所期盼的Plugin,虽然,此Plugin非彼Plugin,但还是值得学习下(毕竟源码只有那么几行).  
### (2). 定义接口
```
package org.springframework.plugin.core;


public interface SamplePlugin extends Plugin<String> {
	void pluginMethod();
}
```
### (3). 定义接口的实现
```
package org.springframework.plugin.core;


public class SamplePluginImplementation implements SamplePlugin {

	public boolean supports(String delimiter) {
		return "FOO".equals(delimiter);
	}

	public void pluginMethod() {

	}
}
```
### (4). 测试
```
package org.springframework.plugin.core.config;

import static org.assertj.core.api.Assertions.*;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.plugin.core.Plugin;
import org.springframework.plugin.core.PluginRegistry;
import org.springframework.plugin.core.SamplePlugin;
import org.springframework.plugin.core.SamplePluginImplementation;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit.jupiter.SpringExtension;


@ContextConfiguration
@ExtendWith(SpringExtension.class)
class EnablePluginRegistriesIntegrationTest {
	
	// *********************************************************
	// 定义一个接口
	// *********************************************************
	@Qualifier("anotherPlugin")
	interface AnotherPlugin extends Plugin<String> {}
	
	// *********************************************************
	// 定义接口的实现
	// *********************************************************
	static class AnotherSamplePluginImplementation implements AnotherPlugin {
	
		@Override
		public boolean supports(String delimiter) {
			return true;
		}
	}
	

	@Configuration
	// *********************************************************
	// 启用插件
	// *********************************************************
	@EnablePluginRegistries({ SamplePlugin.class, AnotherPlugin.class })
	static class Config {

		@Bean
		public SamplePluginImplementation pluginImpl() {
			return new SamplePluginImplementation();
		}
	}

    // *********************************************************
	// 注入插件仓库
	// *********************************************************
	@Autowired 
	private PluginRegistry<SamplePlugin, String> registry;

	// *********************************************************
	// 注入插件仓库
	// *********************************************************
	@Autowired 
	@Qualifier("anotherPlugin") 
	private PluginRegistry<AnotherPlugin,String> anotherPlugin;

	@Test
	void registersPluginRegistries() {
		// *********************************************************
		// 验证
		// *********************************************************
		assertThat(registry).isNotNull();
		assertThat(anotherPlugin).isNotNull();
	}
}
```
### (5). 总结
我觉得唯一不足的地方在于,需要强制使用:PluginRegistry.  