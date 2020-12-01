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
### (3). 简单查询
```
package help.lixin.solrj;

import java.util.List;
import java.util.stream.Stream;

import org.apache.solr.client.solrj.SolrQuery;
import org.apache.solr.client.solrj.impl.HttpSolrClient;
import org.apache.solr.client.solrj.response.QueryResponse;
import org.apache.solr.common.params.SolrParams;

import help.lixin.solrj.pojo.Product;

public class SimpleQueryTest {
	public static void main(String[] args) throws Exception {
		String url = "http://localhost:8080/solr/core_example";
		HttpSolrClient client = new HttpSolrClient.Builder(url).build();
		// 模糊查询(Field):prod_pname的内容
		SolrParams params = new SolrQuery("prod_pname:测试");
		QueryResponse response = client.query(params);
		List<Product> products = response.getBeans(Product.class);
		Stream.of(products).forEach(p->{
			System.out.println(p);
		});

		client.commit();
		client.close();
	}
}
```

### (4). 复杂查询
```
package help.lixin.solrj;

import java.util.List;
import java.util.stream.Stream;

import org.apache.solr.client.solrj.SolrQuery;
import org.apache.solr.client.solrj.impl.HttpSolrClient;
import org.apache.solr.client.solrj.response.QueryResponse;
import org.apache.solr.common.StringUtils;

import help.lixin.solrj.pojo.Product;

public class ComplexQuery {

	public static void main(String[] args) throws Exception {
		String url = "http://localhost:8080/solr/core_example";
		HttpSolrClient client = new HttpSolrClient.Builder(url).build();

		// 商品名称(like)
		String keyWorld = "prod_pname:手机";
		if (StringUtils.isEmpty(keyWorld)) {
			keyWorld = "*:*";
		}

		// 商品类别(fq:equals)
		String prodCatalogName = "prod_catalog_name:手机饰品";

		// 价格(between)
		int startPrice = 0;
		int endPrice = 200;

		// prod_price:[1 TO 100]
		// prod_price:[1 TO * ]
		// prod_price:[* TO 100 ]
		String prodPrice = "prod_price:[" + startPrice + " TO " + endPrice + "]";
		SolrQuery params = new SolrQuery();
		// 设置:q
		params.set("q", keyWorld);

		if (!StringUtils.isEmpty(prodCatalogName)) {
			params.addFilterQuery(prodCatalogName);
		}

		if (!StringUtils.isEmpty(prodPrice)) {
			params.addFilterQuery(prodPrice);
		}

		// 排序
		// prod_price asc,id desc
		params.addSort("prod_price", SolrQuery.ORDER.desc);
		
		// 分页
		params.setStart(0);
		params.setRows(20);

		QueryResponse response = client.query(params);
		List<Product> products = response.getBeans(Product.class);
		System.err.println(products.size());
		Stream.of(products).forEach(p -> {
			System.out.println(p);
		});

		client.commit();
		client.close();
	}
}

```

