---
layout: post
title: 'Spring Data Elasticsearch简单入门' 
date: 2022-01-02
author: 李新
tags:  Spring-Data-Elasticsearch
---

### (1). 概述
Spring Data Elasticsearch在Elasticsearch SDK的基础上,提供了更高级的封装,不过,因为Elasticsearch经常的变化,发现:Spring Data Elasticsearch的API也是经常变的,所以,所谓的架构前瞻性的设计有时是很难跟上变化的,特别是Spring这种做粘贴层的,既要抽出共性,又要考虑个性,还要定义规范.      
在这里对Spring Data Elasticsearch进行一个简单的入门案例,原因是:Elasticsearch SDK自己要封装的还是有点多(比如:Mapping管理/分页管理).    

### (2). 项目结构
```
lixin-macbook:elasticsearch-demo lixin$ tree
.
├── pom.xml
├── src
│   └── main
│       ├── java
│       │   └── help
│       │       └── lixin
│       │           └── demo
│       │               ├── DemoApplication.java
│       │               ├── controller
│       │               │   └── BookController.java
│       │               ├── dao
│       │               │   └── BookRepository.java
│       │               └── model
│       │                   └── Book.java
│       └── resources
│           └── application.yml
└── target
```
### (3). application.yml
```
spring:
  data:
    elasticsearch:
	  # 更多参数请参考这个类:  
	  # org.springframework.boot.autoconfigure.elasticsearch.ElasticsearchProperties
	  # es的集群名称
      cluster-name: docker-cluster
	  # es的集群机器列表
      cluster-nodes: 127.0.0.1:9200
```
### (4). Book
```
package help.lixin.demo.model;

import org.springframework.data.annotation.Id;
import org.springframework.data.elasticsearch.annotations.Document;
import org.springframework.data.elasticsearch.annotations.Field;
import org.springframework.data.elasticsearch.annotations.FieldType;

@Document(indexName = "books")
public class Book {
    @Id
    private String id;

    @Field(type = FieldType.Text)
    private String name;

    @Field(type = FieldType.Text)
    private String summary;

    @Field(type = FieldType.Integer)
    private Integer price;

    public String getId() {
        return id;
    }

    public void setId(String id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getSummary() {
        return summary;
    }

    public void setSummary(String summary) {
        this.summary = summary;
    }

    public Integer getPrice() {
        return price;
    }

    public void setPrice(Integer price) {
        this.price = price;
    }
}
```
### (5). BookRepository
```
package help.lixin.demo.dao;

import help.lixin.demo.model.Book;
import org.springframework.data.elasticsearch.repository.ElasticsearchRepository;

import java.util.List;

public interface BookRepository extends ElasticsearchRepository<Book, String> {
    List<Book> findByNameAndPrice(String name, Integer price);
}
```
### (6). BookController 
```
package help.lixin.demo.controller;

import help.lixin.demo.dao.BookRepository;
import help.lixin.demo.model.Book;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.elasticsearch.core.*;
import org.springframework.data.elasticsearch.core.mapping.IndexCoordinates;
import org.springframework.data.elasticsearch.core.query.Criteria;
import org.springframework.data.elasticsearch.core.query.CriteriaQuery;
import org.springframework.data.elasticsearch.core.query.Query;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.stream.Collectors;

@RestController
public class BookController {
    private ElasticsearchOperations elasticsearchOperations;
    private BookRepository bookRepository;

    public BookController(ElasticsearchOperations elasticsearchOperations, BookRepository bookRepository) {
        this.elasticsearchOperations = elasticsearchOperations;
        this.bookRepository = bookRepository;
    }

    /**
     * @return
     */
    @GetMapping("/mapping")
    public String index() {
        IndexOperations indexOperations = elasticsearchOperations.indexOps(Book.class);
        indexOperations.createMapping(Book.class);
        indexOperations.putMapping(Book.class);
        return "SUCCESS";
    }

    /**
     * curl -X POST -H "Accept: application/json" -H "Content-type: application/json" -d '{ "id":"1" ,"name":"hello","summary":"test","price":80 }'   'http://127.0.0.1:8080/book'
     * curl -X POST -H "Accept: application/json" -H "Content-type: application/json" -d '{ "id":"2" ,"name":"world","summary":"test2","price":90 }'   'http://127.0.0.1:8080/book'
     * curl -X POST -H "Accept: application/json" -H "Content-type: application/json" -d '{ "id":"3" ,"name":"hello","summary":"test","price":60 }'   'http://127.0.0.1:8080/book'
     * curl -X POST -H "Accept: application/json" -H "Content-type: application/json" -d '{ "id":"4" ,"name":"hello","summary":"test","price":70 }'   'http://127.0.0.1:8080/book'
     *
     * @param book
     * @return
     */
    @PostMapping("/book")
    public String save(@RequestBody Book book) {
        Book bookTemp = elasticsearchOperations.save(book);
        return bookTemp.getId();
    }

    /**
     * curl -X GET http://localhost:8080/book/1
     *
     * @param id
     * @return
     */
    @GetMapping("/book/{id}")
    public Book findById(@PathVariable("id") String id) {
        Book book = elasticsearchOperations.get(id, Book.class, IndexCoordinates.of("books"));
        return book;
    }

    /**
     * curl -X GET 'http://localhost:8080/query/hello'
     *
     * @param name
     * @return
     */
    @GetMapping("/query/{name}")
    public List<Book> query(@PathVariable("name") String name) {
        List<Book> result = bookRepository.findByNameAndPrice(name, 80);
        return result;
    }

    /**
     * curl -X GET 'http://localhost:8080/search/hello'
     *
     * @param name
     * @return
     */
    @GetMapping("/search/{name}")
    public List<Book> search(@PathVariable("name") String name) {
        // Criteria criteria = new Criteria("name").is("hello").and("price").greaterThan(30);
        Criteria criteria = new Criteria("name").is(name);
        Query criteriaQuery = new CriteriaQuery(criteria);
        SearchHits<Book> search = elasticsearchOperations.search(criteriaQuery, Book.class);
        List<Book> list = search.getSearchHits().stream().map(item -> item.getContent()).collect(Collectors.toList());
        return list;
    }

    /**
     * curl http://localhost:8080/page/hello
     *
     * @param name
     * @return
     */
    @GetMapping("/page/{name}")
    public Page<Book> page(@PathVariable("name") String name) {
        Criteria criteria = new Criteria("name").is(name);
        Query criteriaQuery = new CriteriaQuery(criteria);
        SearchHits<Book> searchHits = elasticsearchOperations.search(criteriaQuery, Book.class);
        SearchPage<Book> searchPage = SearchHitSupport.searchPageFor(searchHits, Pageable.ofSize(1));
        Page<Book> page = (Page<Book>) SearchHitSupport.unwrapSearchHits(searchPage);
        // 总页数
        int totalPages = page.getTotalPages();
        // 总记录数
        long totalElements = page.getTotalElements();
        // 每页显示多少条
        int size = page.getSize();
        return page;
    }
}
```
### (7). pom.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>help.lixin.elastic.search</groupId>
    <artifactId>elastic-search-parent</artifactId>
    <packaging>jar</packaging>
    <version>1.0.0-SNAPSHOT</version>
    <name>elastic-search-parent</name>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
        <mybatis-plus-boot-starter.version>3.4.2</mybatis-plus-boot-starter.version>

        <spring-cloud.version>2021.0.0</spring-cloud.version>
        <spring-boot-dependencies.version>2.6.1</spring-boot-dependencies.version>
        <spring-cloud-netflix.version>3.1.0</spring-cloud-netflix.version>
        <spring-cloud-alibaba-dependencies>2021.1</spring-cloud-alibaba-dependencies>

        <!--        <spring-cloud.version>Hoxton.RELEASE</spring-cloud.version>-->
        <!--        <spring-boot-dependencies.version>2.2.5.RELEASE</spring-boot-dependencies.version>-->
        <!--        <spring-cloud-netflix.version>2.2.0.RELEASE</spring-cloud-netflix.version>-->
        <!--        <spring-cloud-alibaba-dependencies>2.2.0.RELEASE</spring-cloud-alibaba-dependencies>-->

        <swagger-annotations.version>1.5.13</swagger-annotations.version>
        <gerp-common.version>1.0.0-SNAPSHOT</gerp-common.version>
        <org.mapstruct>1.4.2.Final</org.mapstruct>
        <xxl-job-integration.version>1.0.0-SNAPSHOT</xxl-job-integration.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring-boot-dependencies.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-netflix</artifactId>
                <version>${spring-cloud-netflix.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>com.alibaba.cloud</groupId>
                <artifactId>spring-cloud-alibaba-dependencies</artifactId>
                <version>${spring-cloud-alibaba-dependencies}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
        </dependency>


        <!--  引入ribbon
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
        </dependency>
        -->

        <!--  引入openfeign  -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>
        <!-- 引入nacos -->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
        </dependency>


        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>

```
### (8). 总结
总体来说:Spring Data Elasticsearch在Elasticsearch的SDK上进行了一层封装,相对来说操作比较容易,但是,感觉分页还是比较繁琐的.  