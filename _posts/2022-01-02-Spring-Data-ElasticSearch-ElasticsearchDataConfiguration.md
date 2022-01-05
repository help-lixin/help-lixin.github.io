---
layout: post
title: 'Spring Data Elasticsearch源码之ElasticsearchDataConfiguration$ElasticsearchRestTemplate(三)' 
date: 2022-01-02
author: 李新
tags:  Spring-Data-Elasticsearch
---

### (1). 概述
在这一篇,主要分析下ElasticsearchRestTemplate的初始化,因为,Spring里大量地方都用到了:ElasticsearchRestTemplate

### (2). ElasticsearchRestTemplate初始化
```
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(RestHighLevelClient.class)
static class RestClientConfiguration {
	
	@Bean
	@ConditionalOnMissingBean(value = ElasticsearchOperations.class, name = "elasticsearchTemplate")
	@ConditionalOnBean(RestHighLevelClient.class)
	ElasticsearchRestTemplate elasticsearchTemplate(RestHighLevelClient client, ElasticsearchConverter converter) {
		return new ElasticsearchRestTemplate(client, converter);
	} // end elasticsearchTemplate
	
}
```
### (3). 总结
比较简单的理解就是:ElasticsearchRestTemplate是对ES客户端的包装. 
