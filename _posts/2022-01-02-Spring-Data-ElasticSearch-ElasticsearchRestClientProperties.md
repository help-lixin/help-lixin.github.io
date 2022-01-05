---
layout: post
title: 'Spring Data Elasticsearch源码之ElasticsearchRestClientProperties(一)' 
date: 2022-01-02
author: 李新
tags:  Spring-Data-Elasticsearch
---

### (1). 概述
入门Spring Data ElasticSearch的第一步是要了解配置,所以,需要知道ES的配置类.

### (2). ElasticsearchRestClientProperties
```
package org.springframework.boot.autoconfigure.elasticsearch;

@ConfigurationProperties(prefix = "spring.elasticsearch.rest")
public class ElasticsearchRestClientProperties {
	
	private List<String> uris = new ArrayList<>(Collections.singletonList("http://localhost:9200"));

	private String username;
	
	private String password;
	
	private Duration connectionTimeout = Duration.ofSeconds(1);

	private Duration readTimeout = Duration.ofSeconds(30);
}
```
### (3). 总结
Spring Data ElasticSearch与ElasticSearch进行整合时,配置比较简单.
