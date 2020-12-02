---
layout: post
title: 'Lucene IndexWriter索引(二)'
date: 2018-09-10
author: 李新
tags: Lucene
---


### (1). 通过Lucene建立索引
> 通过Lucene演示,如何建立索引

### (2). pom.xml
```
<project xmlns="http://maven.apache.org/POM/4.0.0"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>help.lixin.lucene</groupId>
	<artifactId>lucene-demo</artifactId>
	<version>1.0.0-SNAPSHOT</version>
	<packaging>jar</packaging>

	<name>lucene-demo</name>
	<url>http://maven.apache.org</url>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<maven.compiler.source>1.8</maven.compiler.source>
		<maven.compiler.target>1.8</maven.compiler.target>
		<java.version>1.8</java.version>
	</properties>

	<dependencies>
		<dependency>
			<groupId>commons-io</groupId>
			<artifactId>commons-io</artifactId>
			<version>2.6</version>
		</dependency>

		<dependency>
			<groupId>org.apache.lucene</groupId>
			<artifactId>lucene-core</artifactId>
			<version>7.7.3</version>
		</dependency>
		
		<dependency>
			<groupId>org.apache.lucene</groupId>
			<artifactId>lucene-analyzers-common</artifactId>
			<version>7.7.3</version>
		</dependency>
		<dependency>
			<groupId>org.apache.lucene</groupId>
			<artifactId>lucene-queryparser</artifactId>
			<version>7.7.3</version>
		</dependency>

		<dependency>
			<groupId>mysql</groupId>
			<artifactId>mysql-connector-java</artifactId>
			<version>5.1.42</version>
		</dependency>
		<dependency>
            <groupId>com.janeluo</groupId>
            <artifactId>ikanalyzer</artifactId>
            <version>2012_u6</version>
        </dependency>
		<dependency>
			<groupId>junit</groupId>
			<artifactId>junit</artifactId>
			<version>4.12</version>
			<scope>test</scope>
		</dependency>
	</dependencies>
</project>

```
### (3). IProductDao
```
package help.lixin.lucene.dao;

import java.util.List;

import help.lixin.lucene.model.Product;

public interface IProductDao {
	List<Product> products();
}

```
### (4). ProductDaoImpl
```
package help.lixin.lucene.dao.impl;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.util.ArrayList;
import java.util.Date;
import java.util.List;

import help.lixin.lucene.dao.IProductDao;
import help.lixin.lucene.model.Product;

public class ProductDaoImpl implements IProductDao {

	private String sql = new StringBuilder() //
			.append(" SELECT  ") //
			.append(" pid,  ") //
			.append(" pname,  ") //
			.append(" catalog,  ") //
			.append(" catalog_name,  ") //
			.append(" price,  ") //
			.append(" number,  ") //
			.append(" description,  ") //
			.append(" picture,  ") //
			.append(" release_time  ") //
			.append(" FROM products  ") //
			.toString();

	@Override
	public List<Product> products() {
		List<Product> products = new ArrayList<Product>(0);
		try {
			Class.forName("com.mysql.jdbc.Driver");
			Connection conn = DriverManager.getConnection("jdbc:mysql://127.0.0.1:3306/solr?characterEncoding=UTF-8",
					"root", "123456");
			PreparedStatement ps = conn.prepareStatement(sql);
			ResultSet rs = ps.executeQuery();
			while (rs.next()) {
				Product product = new Product();
				product.setPid(rs.getInt("pid"));
				product.setPname(rs.getString("pname"));
				product.setCatalog(rs.getInt("catalog"));
				product.setCatalogName(rs.getString("catalog_name"));
				product.setPrice(rs.getDouble("price"));
				product.setNumber(rs.getInt("number"));
				product.setDescription(rs.getString("description"));
				product.setPicture(rs.getString("picture"));
				product.setReleaseTime(new Date(rs.getDate("release_time").getTime()));
				products.add(product);
			}
		} catch (

		Exception e) {
		}
		return products;
	}
}

```
### (5). Product
```
package help.lixin.lucene.model;

import java.io.Serializable;
import java.util.Date;

public class Product implements Serializable {
	private static final long serialVersionUID = 202399649859322307L;
	private Integer pid;
	private String pname;
	private Integer catalog;
	private String catalogName;
	private Double price;
	private Integer number;
	private String description;
	private String picture;
	private Date releaseTime;

	public Integer getPid() {
		return pid;
	}

	public void setPid(Integer pid) {
		this.pid = pid;
	}

	public String getPname() {
		return pname;
	}

	public void setPname(String pname) {
		this.pname = pname;
	}

	public Integer getCatalog() {
		return catalog;
	}

	public void setCatalog(Integer catalog) {
		this.catalog = catalog;
	}

	public String getCatalogName() {
		return catalogName;
	}

	public void setCatalogName(String catalogName) {
		this.catalogName = catalogName;
	}

	public Double getPrice() {
		return price;
	}

	public void setPrice(Double price) {
		this.price = price;
	}

	public Integer getNumber() {
		return number;
	}

	public void setNumber(Integer number) {
		this.number = number;
	}

	public String getDescription() {
		return description;
	}

	public void setDescription(String description) {
		this.description = description;
	}

	public String getPicture() {
		return picture;
	}

	public void setPicture(String picture) {
		this.picture = picture;
	}

	public Date getReleaseTime() {
		return releaseTime;
	}

	public void setReleaseTime(Date releaseTime) {
		this.releaseTime = releaseTime;
	}

	@Override
	public String toString() {
		return "Product [pid=" + pid + ", pname=" + pname + ", catalog=" + catalog + ", catalogName=" + catalogName
				+ ", price=" + price + ", number=" + number + ", description=" + description + ", picture=" + picture
				+ ", releaseTime=" + releaseTime + "]";
	}

}
```
### (6). IndexWriterTest
```
package help.lixin.lucene.service;

import java.nio.file.Paths;
import java.util.ArrayList;
import java.util.List;

import org.apache.lucene.analysis.Analyzer;
import org.apache.lucene.analysis.standard.StandardAnalyzer;
import org.apache.lucene.document.Document;
import org.apache.lucene.document.Field.Store;
import org.apache.lucene.document.TextField;
import org.apache.lucene.index.IndexWriter;
import org.apache.lucene.index.IndexWriterConfig;
import org.apache.lucene.store.Directory;
import org.apache.lucene.store.FSDirectory;
import org.junit.Before;
import org.junit.Test;

import help.lixin.lucene.dao.IProductDao;
import help.lixin.lucene.dao.impl.ProductDaoImpl;
import help.lixin.lucene.model.Product;

public class IndexWriterTest {

	private IProductDao productDao = null;

	@Before
	public void before() {
		productDao = new ProductDaoImpl();
	}

	@Test
	public void testWriter() throws Exception {
		// 1. 采集数据
		List<Product> products = productDao.products();

		// 2. 创建文档对象
		List<Document> documents = new ArrayList<Document>(0);
		for (Product product : products) {
			Document document = buildDocument(product);
			documents.add(document);
		}

		// 3. 创建分词器(标准分词器,对英文分词效果后,对:中文是单字分词)
		Analyzer analyzer = new StandardAnalyzer();

		// 4. 创建索引库存放目录
		String indexDir = "/Users/lixin/Workspace/lucene-demo/indexDir";
		Directory directory = FSDirectory.open(Paths.get(indexDir));

		// 5. 创建IndexWriterConfig,指定:分词器
		IndexWriterConfig config = new IndexWriterConfig(analyzer);

		// 6. 创建IndexWriter,指定:索引目录和中文分词器
		IndexWriter writer = new IndexWriter(directory, config);

		// 7. 写入文档
		writer.addDocuments(documents);
		
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
		document.add(new TextField("pid", String.valueOf(product.getPid()), Store.YES));
		document.add(new TextField("pname", String.valueOf(product.getPname()), Store.YES));
		document.add(new TextField("catalog", String.valueOf(product.getCatalog()), Store.YES));
		document.add(new TextField("catalogName", String.valueOf(product.getCatalogName()), Store.YES));
		document.add(new TextField("price", String.valueOf(product.getPrice()), Store.YES));
		document.add(new TextField("number", String.valueOf(product.getNumber()), Store.YES));
		document.add(new TextField("description", String.valueOf(product.getDescription()), Store.YES));
		document.add(new TextField("picture", String.valueOf(product.getPicture()), Store.YES));
		document.add(new TextField("releaseTime", String.valueOf(product.getReleaseTime().getTime()), Store.YES));
		return document;
	}
}
```
### (7). 脚本

["products脚本"](/assets/lucene/products.sql)

### (8). 执行后索引目录如下

!["IndexWriter"](/assets/lucene/imgs/index-writer-path.jpg)