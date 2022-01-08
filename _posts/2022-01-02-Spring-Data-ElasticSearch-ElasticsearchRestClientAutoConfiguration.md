---
layout: post
title: 'Spring Data Elasticsearch源码之ElasticsearchRestClientAutoConfiguration(二)' 
date: 2022-01-02
author: 李新
tags:  Spring-Data-Elasticsearch
---

### (1). 概述
前面了解到:ElasticsearchRestClientProperties类主要用于配置ES连接的信息,下一步,就应该要分析,配置信息是如何转换成ES的Client的.  

### (2). ElasticsearchRestClientAutoConfiguration
```
package org.springframework.boot.autoconfigure.elasticsearch;

@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(RestHighLevelClient.class)
@ConditionalOnMissingBean(RestClient.class)
@EnableConfigurationProperties(ElasticsearchRestClientProperties.class)
public class ElasticsearchRestClientAutoConfiguration {

	
	@Configuration(proxyBeanMethods = false)
	@ConditionalOnMissingBean(RestClientBuilder.class)
	static class RestClientBuilderConfiguration {
		
		// 1. RestClientBuilderCustomizer
		@Bean
		RestClientBuilderCustomizer defaultRestClientBuilderCustomizer(ElasticsearchRestClientProperties properties) {
			return new DefaultRestClientBuilderCustomizer(properties);
		}

		// 2. 创建RestClientBuilder
		@Bean
		RestClientBuilder elasticsearchRestClientBuilder(ElasticsearchRestClientProperties properties,
				ObjectProvider<RestClientBuilderCustomizer> builderCustomizers) {
			HttpHost[] hosts = properties.getUris().stream().map(this::createHttpHost).toArray(HttpHost[]::new);
			RestClientBuilder builder = RestClient.builder(hosts);
			builder.setHttpClientConfigCallback((httpClientBuilder) -> {
				builderCustomizers.orderedStream().forEach((customizer) -> customizer.customize(httpClientBuilder));
				return httpClientBuilder;
			});
			builder.setRequestConfigCallback((requestConfigBuilder) -> {
				builderCustomizers.orderedStream().forEach((customizer) -> customizer.customize(requestConfigBuilder));
				return requestConfigBuilder;
			});
			builderCustomizers.orderedStream().forEach((customizer) -> customizer.customize(builder));
			return builder;
		}
	} // end RestClientBuilderConfiguration


	@Configuration(proxyBeanMethods = false)
	@ConditionalOnMissingBean(RestHighLevelClient.class)
	static class RestHighLevelClientConfiguration {
		
		// 3. 基于:RestClientBuilder创建:RestHighLevelClient
		@Bean
		RestHighLevelClient elasticsearchRestHighLevelClient(RestClientBuilder restClientBuilder) {
			return new RestHighLevelClient(restClientBuilder);
		}

	} // end RestHighLevelClientConfiguration
	
}
```
### (3). 总结
ElasticsearchRestClientAutoConfiguration的主要目的就是基于ElasticsearchRestClientProperties和RestClientBuilder创建RestHighLevelClient对象.
