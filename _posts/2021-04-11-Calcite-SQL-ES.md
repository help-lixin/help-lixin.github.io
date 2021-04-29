---
layout: post
title: 'Calcite 通过SQL读取ES案例入门(二)'
date: 2021-04-11
author: 李新
tags:  Calcite
---

### (1). 前言
> 想实现这样的功能,通过编写SQL,即可实现对ES文档进行检索. 

### (2). 在ES中建立索引库(books)
> 请参考这篇文章

["books索引"](/2020/11/29/ElasticSearch-Operator.html)

### (3). ElasticDemo

```
package help.lixin.calcite;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.ResultSet;
import java.sql.Statement;
import java.util.Properties;

import org.apache.calcite.adapter.elasticsearch.ElasticsearchSchema;
import org.apache.calcite.jdbc.CalciteConnection;
import org.apache.calcite.schema.SchemaPlus;
import org.apache.http.HttpHost;
import org.elasticsearch.client.RestClient;

import com.fasterxml.jackson.databind.ObjectMapper;

public class ElasticDemo {
	public static void main(String[] args) throws Exception {
		// 1.构建ElasticsearchSchema对象,在Calcite中,不同数据源对应不同Schema,比如:CsvSchema、DruidSchema、ElasticsearchSchema等
		RestClient restClient = RestClient.builder(new HttpHost("localhost", 9200)).build();
		// 指定索引库
		ElasticsearchSchema elasticsearchSchema = new ElasticsearchSchema(restClient, new ObjectMapper(), "books");

		// 2.构建Connection
		// 2.1 设置连接参数
		Properties info = new Properties();
		// 不区分sql大小写
		info.setProperty("caseSensitive", "false");
		
		// 2.2 获取标准的JDBC Connection
		Connection connection = DriverManager.getConnection("jdbc:calcite:", info);
		// 2.3 获取Calcite封装的Connection
		CalciteConnection calciteConnection = connection.unwrap(CalciteConnection.class);

		// 3.构建RootSchema，在Calcite中，RootSchema是所有数据源schema的parent，多个不同数据源schema可以挂在同一个RootSchema下
		// 以实现查询不同数据源的目的
		SchemaPlus rootSchema = calciteConnection.getRootSchema();

		// 4.将不同数据源schema挂载到RootSchema，这里添加ElasticsearchSchema
		rootSchema.add("es", elasticsearchSchema);

		// 5.执行SQL查询，通过SQL方式访问object对象实例
		// 条件查询
		// String sql = "SELECT _MAP['id'],_MAP['title'],_MAP['price'] FROM es.books WHERE _MAP['price'] > 60 LIMIT 2";
		// 统计索引数量
		// String sql = "SELECT count(*) FROM es.books WHERE _MAP['price'] > 50 ";
		// 分页查询
		String sql = "SELECT * FROM es.books WHERE _MAP['price'] > 10 offset 0 fetch next 3 rows only";
		Statement statement = calciteConnection.createStatement();
		ResultSet resultSet = statement.executeQuery(sql);

		// 6.遍历打印查询结果集
		System.out.println(ResultSetUtil.resultString(resultSet));
	}
}
```
### (4). ResultSetUtil
```
package help.lixin.calcite;

import java.sql.ResultSet;
import java.sql.ResultSetMetaData;
import java.sql.SQLException;
import java.util.ArrayList;
import java.util.List;
import java.util.stream.Collectors;

public class ResultSetUtil {
	public static String resultString(ResultSet resultSet) throws SQLException {
        return resultString(resultSet, false);
    }

    public static String resultString(ResultSet resultSet, boolean printHeader) throws SQLException {
        List<List<Object>> resultList = resultList(resultSet, printHeader);
        return resultString(resultList);
    }

    public static List<List<Object>> resultList(ResultSet resultSet) throws SQLException {
        return resultList(resultSet, false);
    }

    public static String resultString(List<List<Object>> resultList) throws SQLException {
        StringBuilder builder = new StringBuilder();
        resultList.forEach(row -> {
            String rowStr = row.stream()
                    .map(columnValue -> columnValue + ", ")
                    .collect(Collectors.joining());
            rowStr = rowStr.substring(0, rowStr.lastIndexOf(", ")) + "\n";
            builder.append(rowStr);
        });
        return builder.toString();
    }

    public static List<List<Object>> resultList(ResultSet resultSet, boolean printHeader) throws SQLException {
        ArrayList<List<Object>> results = new ArrayList<>();
        final ResultSetMetaData metaData = resultSet.getMetaData();
        final int columnCount = metaData.getColumnCount();
        if (printHeader) {
            ArrayList<Object> header = new ArrayList<>();
            for (int i = 1; i <= columnCount; i++) {
                header.add(metaData.getColumnName(i));
            }
            results.add(header);
        }
        while (resultSet.next()) {
            ArrayList<Object> row = new ArrayList<>();
            for (int i = 1; i <= columnCount; i++) {
                row.add(resultSet.getObject(i));
            }
            results.add(row);
        }
        return results;
    }
}
```
### (5). 查看控制台(验证结果)
```
# SELECT * FROM es.books WHERE _MAP['price'] > 10 offset 0 fetch next 3 rows only
# Calcite针对ES的分页还有一些独特.

{id=1, title=Java编程思想, language=java, author=Bruce Eckel, price=70.2, publish_time=2007-10-01, description=Java学习必读经典,殿堂级著作,赢得了全球程序员的广泛赞誉}
{id=2, title=Java程序性能优化, language=java, author=葛一鸣, price=46.5, publish_time=2012-08-01, description=让你的Java程序更快,更稳定.深入剖析软件层面,代码层面,JVM虚拟机层面的优化方法}
{id=3, title=Python科学计算, language=python, author=张惹愚, price=81.4, publish_time=2016-05-01, description=零基础学Python,光盘中作者独家整合开发winPython环境,涵盖了Python各个扩展库}
```
### (6). pom.xml(依赖文件)
```
<project xmlns="http://maven.apache.org/POM/4.0.0"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>help.lixin.calcite</groupId>
	<artifactId>calcite-example</artifactId>
	<version>1.0.0-SNAPSHOT</version>
	<packaging>jar</packaging>

	<name>calcite-example</name>
	<url>http://maven.apache.org</url>

	<properties>
		<maven.compiler.source>8</maven.compiler.source>
		<maven.compiler.target>8</maven.compiler.target>
		<environment-service.version>1.0.0-SNAPSHOT</environment-service.version>
		<elasticsearch.version>7.1.0</elasticsearch.version>
	</properties>

	<dependencies>
		<dependency>
		    <groupId>org.apache.calcite</groupId>
		    <artifactId>calcite-core</artifactId>
		    <version>1.26.0</version>
		</dependency>
		<dependency>
		    <groupId>org.apache.calcite</groupId>
		    <artifactId>calcite-linq4j</artifactId>
		    <version>1.26.0</version>
		</dependency>
		<dependency>
		  <groupId>org.apache.calcite</groupId>
		  <artifactId>calcite-elasticsearch</artifactId>
		  <version>1.26.0</version>
		</dependency>
		<dependency>
		    <groupId>org.apache.calcite</groupId>
		    <artifactId>calcite-file</artifactId>
		    <version>1.26.0</version>
		</dependency>
		<dependency>
		    <groupId>org.apache.calcite</groupId>
		    <artifactId>calcite-csv</artifactId>
		    <version>1.26.0</version>
		</dependency>
		
		<dependency>
			<groupId>org.elasticsearch.client</groupId>
			<artifactId>elasticsearch-rest-client</artifactId>
			<version>7.1.0</version>
		</dependency>
		<dependency>
			<groupId>org.elasticsearch.client</groupId>
			<artifactId>elasticsearch-rest-high-level-client</artifactId>
			<version>7.1.0</version>
			<!-- 设置排除 -->
			<exclusions>
				<exclusion>
					<groupId>org.elasticsearch</groupId>
					<artifactId>elasticsearch</artifactId>
				</exclusion>
				<exclusion>
					<groupId>org.elasticsearch</groupId>
					<artifactId>elasticsearch-core</artifactId>
				</exclusion>
				<exclusion>
					<groupId>org.elasticsearch</groupId>
					<artifactId>elasticsearch-cli</artifactId>
				</exclusion>
				<exclusion>
					<groupId>org.elasticsearch</groupId>
					<artifactId>elasticsearch-secure-sm</artifactId>
				</exclusion>
				<exclusion>
					<groupId>org.elasticsearch</groupId>
					<artifactId>elasticsearch-x-content</artifactId>
				</exclusion>
			</exclusions>
		</dependency>
		
		<!-- 重新添加指定版本 -->
		<dependency>
			<groupId>org.elasticsearch</groupId>
			<artifactId>elasticsearch</artifactId>
			<version>${elasticsearch.version}</version>
		</dependency>
		<dependency>
			<groupId>org.elasticsearch</groupId>
			<artifactId>elasticsearch-core</artifactId>
			<version>${elasticsearch.version}</version>
		</dependency>
		
		<dependency>
			<groupId>org.elasticsearch</groupId>
			<artifactId>elasticsearch-cli</artifactId>
			<version>${elasticsearch.version}</version>
		</dependency>
		<dependency>
			<groupId>org.elasticsearch</groupId>
			<artifactId>elasticsearch-secure-sm</artifactId>
			<version>${elasticsearch.version}</version>
		</dependency>
		<dependency>
			<groupId>org.elasticsearch</groupId>
			<artifactId>elasticsearch-x-content</artifactId>
			<version>${elasticsearch.version}</version>
		</dependency>
		
		<dependency>
		    <groupId>junit</groupId>
		    <artifactId>junit</artifactId>
		    <version>4.11</version>
		</dependency>
	</dependencies>
</project>
```
### (7). 总结
> 通过编写SQL,即可实现对ES的检索.