---
layout: post
title: 'MongoDB Spring Data MongoDB基本使用(四)'
date: 2021-05-24
author: 李新
tags:  MongoDB
---

### (1). 概述
> 前面搭建了一个MongoDB的分片集群,在这里,通过spring-data-mongodb连接mongs,进行基本的操作.  

### (2). 项目结构如下
```
lixin-macbook:mongodb-example lixin$ tree
.
├── pom.xml
├── src
│   ├── main
│   │   ├── java
│   │   │   └── help
│   │   │       └── lixin
│   │   │           └── mongodb
│   │   │               ├── App.java
│   │   │               ├── entity
│   │   │               │   └── User.java
│   │   │               └── service
│   │   │                   ├── IUserService.java
│   │   │                   └── impl
│   │   │                       └── UserService.java
│   │   └── resources
│   │       └── application.properties
│   └── test
│       └── java
│           └── help
│               └── lixin
│                   └── mongodb
│                       └── IUserServiceTest.java
└── target
```
### (3). 实体对象(User)

```
package help.lixin.mongodb.entity;

import java.io.Serializable;

import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;
import org.springframework.data.mongodb.core.mapping.Field;

@Document("users")
public class User implements Serializable {
	private static final long serialVersionUID = 4566848882330976364L;
	@Id
	private String id;
	@Field("name")
	private String name;
	@Field("age")
	private Integer age;
	@Field("gender")
	private String gender;

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

	public Integer getAge() {
		return age;
	}

	public void setAge(Integer age) {
		this.age = age;
	}

	public String getGender() {
		return gender;
	}

	public void setGender(String gender) {
		this.gender = gender;
	}

	@Override
	public String toString() {
		return "User [name=" + name + ", age=" + age + ", gender=" + gender + "]";
	}
}
```
### (4). 接口(IUserService)
```
package help.lixin.mongodb.service;

import java.util.List;

import help.lixin.mongodb.entity.User;

public interface IUserService {
	User save(User user);

	Boolean update(User user);

	List<User> findAll();
	
	Boolean delete(User user);
}
```
### (5). 接口实现(UserService)
> 在这里用的是:MongoTemplate,因为:MongoRepository在分片的情况下做update时,抛出了异常.  

```
package help.lixin.mongodb.service.impl;

import java.util.List;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.data.mongodb.core.query.Criteria;
import org.springframework.data.mongodb.core.query.Query;
import org.springframework.data.mongodb.core.query.Update;
import org.springframework.stereotype.Service;

import com.mongodb.client.result.DeleteResult;
import com.mongodb.client.result.UpdateResult;

import help.lixin.mongodb.entity.User;
import help.lixin.mongodb.service.IUserService;

/**
 * 强烈不要用:MongoRepository,MongoDB在单机(以及副本集)情况下都没问题,但是,在Collection分片集群的情况下,会抛出如下异常:<br/>
 * org.springframework.dao.DataIntegrityViolationException: Failed to target
 * upsert by query :: could not extract exact shard key; nested exception is
 * com.mongodb.MongoWriteException: Failed to target upsert by query :: could
 * not extract exact shard key <br/>

 * @author lixin
 */
@Service
public class UserService implements IUserService {

	@Autowired
	private MongoTemplate mongoTemplate;

	@Override
	public User save(User user) {
		return mongoTemplate.insert(user);
	}

	@Override
	public Boolean update(User user) {
		Query query = Query.query(Criteria.where("_id").is(user.getId()));
		// { "$set" : { "age" : 100, "gender" : "男" } }
		Update update = Update.update("age", user.getAge());
		update.set("gender", user.getGender());

		UpdateResult result = mongoTemplate.updateMulti(query, update, User.class);
		return result.getModifiedCount() > 0 ? true : false;
	}

	public Boolean delete(User user) {
		Query query = Query.query(Criteria.where("name").is(user.getName()));
		DeleteResult result = mongoTemplate.remove(query, User.class);
		return result.getDeletedCount() > 0 ? true : false;
	}

	@Override
	public List<User> findAll() {
		return mongoTemplate.findAll(User.class);
	}
}
```
### (6). application.properties
```
# mongs uri = mongodb://10.211.55.100:27017,10.211.55.100:27018/test1
spring.data.mongodb.uri=mongodb://10.211.55.100:27017,10.211.55.100:27018/test1
```
### (7). 测试类(IUserServiceTest)
```
package help.lixin.mongodb;

import java.util.List;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

import help.lixin.mongodb.entity.User;
import help.lixin.mongodb.service.IUserService;
import junit.framework.Assert;

@SuppressWarnings("deprecation")
@RunWith(SpringRunner.class)
@SpringBootTest
public class IUserServiceTest {

	@Autowired
	private IUserService userService;

	@Test
	public void testSave() throws Exception {
		User user = new User();
		user.setId("hello");
		user.setName("李四");
		user.setAge(18);
		user.setGender("女");
		User persiteEntiry = userService.save(user);
		Assert.assertEquals(user.getId(), persiteEntiry.getId());
	}

	@Test
	public void testUpdate() throws Exception {
		User user = new User();
		user.setId("hello");
		user.setName("李四");
		user.setAge(100);
		user.setGender("男");
		boolean isSucc = userService.update(user);
		Assert.assertEquals(true, isSucc);
	}

	@Test
	public void testDel() throws Exception {
		User user = new User();
		user.setId("hello");
		user.setName("李四");
		user.setAge(100);
		user.setGender("男");
		boolean isSucc = userService.delete(user);
		Assert.assertEquals(true, isSucc);
	}

	@Test
	public void testList() throws Exception {
		List<User> list = userService.findAll();
		Assert.assertEquals(1000, list.size());
	}

}
```
### (8). maven依赖(pom.xml)
```
<project xmlns="http://maven.apache.org/POM/4.0.0"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>help.lixin.mongodb</groupId>
	<artifactId>mongodb-example</artifactId>
	<version>1.0.0-SNAPSHOT</version>
	<packaging>jar</packaging>

	<name>mongodb-example</name>
	<url>http://maven.apache.org</url>

	<properties>
		<maven.compiler.source>8</maven.compiler.source>
		<maven.compiler.target>8</maven.compiler.target>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
	</properties>
	
	<dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>Greenwich.RELEASE</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>2.1.0.RELEASE</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-netflix</artifactId>
                <version>2.1.1.RELEASE</version>
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
            <artifactId>spring-boot-starter-test</artifactId>
        </dependency>
        <dependency>
		    <groupId>org.springframework.data</groupId>
		    <artifactId>spring-data-mongodb</artifactId>
		</dependency>
	</dependencies>
</project>
```
### (9). 总结
> 在使用:spring-data-mongodb时,要注意:在分片的情况下,不要用:MongoRepository.   