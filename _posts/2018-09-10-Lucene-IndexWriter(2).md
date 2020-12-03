---
layout: post
title: 'Lucene IndexWriter索引更新(二)'
date: 2018-09-10
author: 李新
tags: Lucene
---


### (1). 更新Lucene中的Document
> 通过Lucene演示,更新(删除)Lucene中的Document

### (2). IndexUpdateTest
```
package help.lixin.lucene.service;

import java.nio.file.Paths;
import java.util.Date;

import org.apache.lucene.analysis.Analyzer;
import org.apache.lucene.analysis.standard.StandardAnalyzer;
import org.apache.lucene.document.Document;
import org.apache.lucene.document.DoublePoint;
import org.apache.lucene.document.Field.Store;
import org.apache.lucene.document.IntPoint;
import org.apache.lucene.document.LongPoint;
import org.apache.lucene.document.StoredField;
import org.apache.lucene.document.StringField;
import org.apache.lucene.document.TextField;
import org.apache.lucene.index.IndexWriter;
import org.apache.lucene.index.IndexWriterConfig;
import org.apache.lucene.index.Term;
import org.apache.lucene.store.Directory;
import org.apache.lucene.store.FSDirectory;
import org.junit.Test;

import help.lixin.lucene.model.Product;

public class IndexUpdateTest {

	@Test
	public void testUpdate() throws Exception {
		Product product = new Product();
		product.setPid(1);
		product.setPname("花儿朵朵彩色金属门后挂&nbsp;8钩免钉门背挂钩2066");
		product.setCatalog(18);
		product.setCatalogName("测试分类名称");
		product.setPrice(19.0);
		product.setNumber(10000);
		product.setPicture("2014032613103438.png");
		product.setDescription("暂无太多描述");
		product.setReleaseTime(new Date());
		
		// 1. 构建需要修改的:Document对象
		Document document = buildDocument(product);
		
		// 3. 创建分词器(标准分词器,对英文分词效果后,对:中文是单字分词)
		Analyzer analyzer = new StandardAnalyzer();
		
		// 4. 创建索引库存放目录
		String indexDir = "/Users/lixin/Workspace/lucene-demo/indexDir";
		Directory directory = FSDirectory.open(Paths.get(indexDir));

		// 5. 创建IndexWriterConfig,指定:分词器
		IndexWriterConfig config = new IndexWriterConfig(analyzer);

		// 6. 创建IndexWriter,指定:索引目录和中文分词器
		IndexWriter writer = new IndexWriter(directory, config);

		// 7. 修改文档
		// term: 修改条件
		// doc:新的document
		long r = writer.updateDocument(new Term("pid", "1"), document);
		
		// 8. 释放资源
		writer.flush();
		writer.close();
	}

	/**
	 * 构建:Document
	 * 
	 * @param product
	 * @return
	 */
	private Document buildDocument(Product product) {
		Document document = new Document();
		/**
		 * 是否分词:N
		 * 是否索引:Y
		 * 是否存储:Y
		 */
		
		document.add(new LongPoint("pid", product.getPid()));
		document.add(new StoredField("pid", product.getPid()));
		
		
		/**
		 * 是否分词:Y
		 * 是否索引:Y
		 * 是否存储:Y
		 */
		document.add(new TextField("pname", product.getPname(), Store.YES));
		
		/**
		 * 是否分词:N
		 * 是否索引:Y
		 * 是否存储:Y
		 */
		document.add(new IntPoint("catalog", product.getCatalog()));
		document.add(new StoredField("catalog", product.getCatalog()));
		
		/**
		 * 是否分词:Y
		 * 是否索引:Y
		 * 是否存储:Y
		 */
		document.add(new TextField("catalogName", product.getCatalogName(), Store.YES));
		
		/**
		 * 是否分词:Y(好像没得给配置,Lucene算法规定了的)
		 * 是否索引:Y
		 * 是否存储:Y
		 */
		document.add(new DoublePoint("price", product.getPrice()));
		document.add(new StoredField("price", product.getPrice()));
		
		/**
		 * 是否分词:N
		 * 是否索引:Y
		 * 是否存储:Y
		 */
		document.add(new IntPoint("number", product.getNumber()));
		document.add(new StoredField("number", product.getNumber()));
		
		/**
		 * 是否分词:Y
		 * 是否索引:Y
		 * 是否存储:N
		 */
		document.add(new TextField("description", String.valueOf(product.getDescription()), Store.NO));
		
		/**
		 * 是否分词:N
		 * 是否索引:Y
		 * 是否存储:Y
		 */
		document.add(new StringField("picture", product.getPicture(), Store.YES));
		
		/**
		 * 是否分词:N
		 * 是否索引:Y
		 * 是否存储:Y
		 */
		document.add(new LongPoint("releaseTime", product.getReleaseTime().getTime()));
		document.add(new StoredField("releaseTime", product.getReleaseTime().getTime()));
		return document;
	}
}
```
### (3). IndexDeleteTest
```
package help.lixin.lucene.service;

import java.nio.file.Paths;

import org.apache.lucene.analysis.Analyzer;
import org.apache.lucene.analysis.standard.StandardAnalyzer;
import org.apache.lucene.index.IndexWriter;
import org.apache.lucene.index.IndexWriterConfig;
import org.apache.lucene.index.Term;
import org.apache.lucene.store.Directory;
import org.apache.lucene.store.FSDirectory;
import org.junit.Test;

public class IndexDeleteTest {

	@Test
	public void testDelete() throws Exception {
		// 3. 创建分词器(标准分词器,对英文分词效果后,对:中文是单字分词)
		Analyzer analyzer = new StandardAnalyzer();
		
		// 4. 创建索引库存放目录
		String indexDir = "/Users/lixin/Workspace/lucene-demo/indexDir";
		Directory directory = FSDirectory.open(Paths.get(indexDir));

		// 5. 创建IndexWriterConfig,指定:分词器
		IndexWriterConfig config = new IndexWriterConfig(analyzer);

		// 6. 创建IndexWriter,指定:索引目录和中文分词器
		IndexWriter writer = new IndexWriter(directory, config);

		// 7. 修改文档
		// term: 修改条件
		// doc:新的document
		long r = writer.deleteDocuments(new Term("pid", "1"));
		
		// 8. 释放资源
		writer.flush();
		writer.close();
	}
}
```
