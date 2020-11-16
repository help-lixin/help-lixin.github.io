---
layout: post
title: 'Solr SolrJ'
date: 2018-10-01
author: 李新
tags: Solr
---

### (1). 保存或更新数据
```
package help.lixin.solrj;

import java.io.IOException;

import org.apache.solr.client.solrj.SolrServerException;
import org.apache.solr.client.solrj.impl.HttpSolrClient;
import org.apache.solr.client.solrj.response.UpdateResponse;

import help.lixin.solrj.pojo.Product;

public class SaveOrUpdateTest {
	public static void main(String[] args) throws IOException, SolrServerException {
		// 请求URL
		String url = "http://localhost:8080/solr/core_example";
		HttpSolrClient client = new HttpSolrClient.Builder(url).build();
		Product product = new Product();
		product.setPid("999999");
		product.setName("后台测试");
		product.setCatalogName("测试");
		product.setDescription("后台测试描述2222");
		product.setPicture("test.png");
		product.setPrice(10.09D);
		UpdateResponse response = client.addBean(product);
		client.commit();
		client.close();
	}
}
```
### (2). 删除
```
package help.lixin.solrj;

import org.apache.solr.client.solrj.impl.HttpSolrClient;
import org.apache.solr.client.solrj.response.UpdateResponse;

public class DelTest {
	public static void main(String[] args) throws Exception {
		// 请求URL
		String url = "http://localhost:8080/solr/core_example";
		HttpSolrClient client = new HttpSolrClient.Builder(url).build();
		UpdateResponse response = client.deleteByQuery("id:999999");
		client.commit();
		client.close();
	}
}
```
### (3). 查询
```

```