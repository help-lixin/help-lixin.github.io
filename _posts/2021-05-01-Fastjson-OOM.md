---
layout: post
title: 'Fastjson OOM'
date: 2021-05-01
author: 李新
tags:  OOM
---

### (1). 前言
> 1. 业务知识如下:  
> 2. HelloController提供http服务,给client调用.
> 3. ClientTest通过OKHttp调用(HelloController),并解析返回内容. 

### (2). HelloController

```
package help.lixin.samples.controller;

import java.util.ArrayList;
import java.util.List;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import help.lixin.samples.common.Result;
import help.lixin.samples.entity.User;

@RestController
public class HelloController {
	private Logger logger = LoggerFactory.getLogger(HelloController.class);

	@GetMapping("/query")
	public Result<List<User>> query() {
		logger.debug("start query");
		User user1 = new User();
		user1.setName("张三");
		user1.setAge(25);

		User user2 = new User();
		user2.setName("李四");
		user2.setAge(28);

		List<User> data = new ArrayList<User>();
		data.add(user1);
		data.add(user2);

		return Result.<List<User>>newBuilder()
		             .code(200)
					 .message("Success")
					 .data(data)
					 .build();
	}
}
```
### (2). 实体对象(User)
```
package help.lixin.samples.entity;

import java.io.Serializable;

public class User implements Serializable {
	private static final long serialVersionUID = -6963110128289327626L;
	private String name;
	private int age;

	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}

	public int getAge() {
		return age;
	}

	public void setAge(int age) {
		this.age = age;
	}

	@Override
	public String toString() {
		return "User [name=" + name + ", age=" + age + "]";
	}
}
```
### (3). 返回结果包装对象(Result)
```
package help.lixin.samples.common;

import java.io.Serializable;
import java.util.HashMap;
import java.util.Map;

/**
 * 返回结果集的封装.
 * 
 * @author lixin
 */
public class Result<T> implements Serializable {
	private static final long serialVersionUID = -8754624853072375484L;
	private int code;
	private String message;
	private T data;
	private Map<Object, Object> others = new HashMap<>();

	public static <T> Builder<T> newBuilder() {
		return new Builder<T>();
	}

	public static class Builder<T> {
		private Result<T> result = new Result<T>();

		public Builder<T> code(int code) {
			this.result.code = code;
			return this;
		}

		public Builder<T> message(String message) {
			this.result.message = message;
			return this;
		}

		public Builder<T> data(T data) {
			this.result.data = data;
			return this;
		}

		@SuppressWarnings({ "rawtypes", "unchecked" })
		public Builder<T> others(Map others) {
			if (null != others && !others.isEmpty()) {
				this.result.others.putAll(others);
			}
			return this;
		}

		public Builder<T> other(Object key, Object value) {
			if (null != key) {
				this.result.others.put(key, value);
			}
			return this;
		}

		public Result<T> build() {
			return result;
		}
	}

	public int getCode() {
		return code;
	}

	public void setCode(int code) {
		this.code = code;
	}

	public String getMessage() {
		return message;
	}

	public void setMessage(String message) {
		this.message = message;
	}

	public T getData() {
		return data;
	}

	public void setData(T data) {
		this.data = data;
	}

	public Map<Object, Object> getOthers() {
		return others;
	}

	public void setOthers(Map<Object, Object> others) {
		this.others = others;
	}

	@Override
	public String toString() {
		return "Result [code=" + code + ", message=" + message + ", data=" + data + ", others=" + others + "]";
	}
}
```
### (4). ClientTest
```
package help.lixin.samples;

import java.lang.reflect.Type;
import java.util.List;

import com.alibaba.fastjson.JSON;

import help.lixin.samples.common.Result;
import help.lixin.samples.entity.User;
import okhttp3.Call;
import okhttp3.OkHttpClient;
import okhttp3.Request;
import okhttp3.Response;

/**
 *  1. 配置最大内存,以及出现OOM时,自动dump <br/>
 * -Xmx10M -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/Users/lixin/Downloads/dump
 * @author lixin
 *
 */
public class ClientTest {
	public static void main(String[] args) throws Exception {
		OkHttpClient client = new OkHttpClient();
		for (;;) {
			Request request = new Request.Builder().url("http://localhost:8080/query").get().build();
			Call call = client.newCall(request);
			Response response = call.execute();
			String body = response.body().string();
			// ******************************************************************
			// 2. 注意:这里使用的是:com.google.gson.reflect.TypeToken
			// ******************************************************************
			Type type = new com.google.gson.reflect.TypeToken<Result<List<User>>>() {}.getType();
			Result<List<User>> result = JSON.parseObject(body, type);
			System.out.println(result);
			Thread.sleep(100);
		}
	}
}
```
### (5). 依赖配置如下(pom.xml)
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>help.lixin</groupId>
	<artifactId>spring-colue-samples-provider3</artifactId>
	<packaging>jar</packaging>
	<version>1.1.0</version>
	<name>spring-cloud-samples-provider3 ${project.version}</name>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<maven.compiler.source>8</maven.compiler.source>
		<maven.compiler.target>8</maven.compiler.target>
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
			<groupId>com.google.code.gson</groupId>
			<artifactId>gson</artifactId>
		</dependency>
		
		<dependency>
			<groupId>com.squareup.okhttp3</groupId>
			<artifactId>okhttp</artifactId>
			<version>3.8.1</version>
		</dependency>
		<dependency>
			<groupId>com.alibaba</groupId>
			<artifactId>fastjson</artifactId>
			<version>1.2.76</version>
		</dependency>
				
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-actuator</artifactId>
		</dependency>
	</dependencies>
</project>
```
### (6). 控制台输出
```
Result [code=200, message=Success, data=[User [name=张三, age=25], User [name=李四, age=28]], others={}]
// *************************************************************
// 出现GC了
// *************************************************************
java.lang.OutOfMemoryError: GC overhead limit exceeded
Dumping heap to /Users/lixin/Downloads/dump/java_pid2483.hprof ...
Heap dump file created [14648286 bytes in 0.122 secs]
*** java.lang.instrument ASSERTION FAILED ***: "!errorOutstanding" with message can't create name string at JPLISAgent.c line: 807
Exception in thread "main" java.lang.OutOfMemoryError: GC overhead limit exceeded
	at java.lang.AbstractStringBuilder.<init>(AbstractStringBuilder.java:68)
	at java.lang.StringBuilder.<init>(StringBuilder.java:89)
Exception in thread "RMI TCP Connection(idle)" java.lang.OutOfMemoryError: GC overhead limit exceeded
Exception in thread "RMI TCP Connection(idle)" java.lang.OutOfMemoryError: Java heap space
Exception in thread "RMI TCP Connection(idle)" java.lang.OutOfMemoryError: Java heap space
```
### (7). jvisualvm监控信息
!["jvisualvm oom "](/assets/oom/imgs/jvisualvm_oom_1.png)
!["jvisualvm oom "](/assets/oom/imgs/jvisualvm_oom_2.png)
!["jvisualvm oom"](/assets/oom/imgs/jvisualvm_oom_3.png)

### (8). eclipse mat排查OOM
!["eclipse mat setting"](/assets/oom/imgs/eclipse-mat-1.png)
!["eclipse mat oom"](/assets/oom/imgs/eclipse-mat-2.png)
!["eclipse mat oom"](/assets/oom/imgs/eclipse-mat-3.png)
!["eclipse mat oom"](/assets/oom/imgs/eclipse-mat-4.png)
!["eclipse mat oom"](/assets/oom/imgs/eclipse-mat-5.jpg)
!["eclipse mat oom"](/assets/oom/imgs/eclipse-mat-6.png)


### (9). jconsole监控OOM
> 当箭头是呈向上的驱势时,代表着存在OOM.  

!["jconsole 监控 oom"](/assets/oom/imgs/jconsole-oom.jpg)

### (10). Bug位置(ParserConfig.getDeserializer)
```
public class ParserConfig {
	// 存放:Type与反序列化的关系.
	private final IdentityHashMap<Type, ObjectDeserializer> deserializers         = new IdentityHashMap<Type, ObjectDeserializer>();

	
	public ObjectDeserializer getDeserializer(Type type) {
		// ... ... 
		} else {
			// 1. 根据clazz创建相应的:BeanDeserializer(JavaBeanDeserializer)
			deserializer = createJavaBeanDeserializer(clazz, type);
		}
		
		// 2. 缓存
		putDeserializer(type, deserializer);
	} // end getDeserializer


	public void putDeserializer(Type type, ObjectDeserializer deserializer) {
		// 
		Type mixin = JSON.getMixInAnnotations(type);
		if (mixin != null) {
			IdentityHashMap<Type, ObjectDeserializer> mixInClasses = this.mixInDeserializers.get(type);
			if (mixInClasses == null) {
				//多线程下可能会重复创建，但不影响正确性
				mixInClasses = new IdentityHashMap<Type, ObjectDeserializer>(4);
				this.mixInDeserializers.put(type, mixInClasses);
			}
			mixInClasses.put(mixin, deserializer);
		} else {
			// *************************************************************************************
			// bug位置,每次都会map里添加新的数据,所以:deserializers会一直增长.
			// *************************************************************************************
			this.deserializers.put(type, deserializer);
		}
	} // end putDeserializer
}
```
### (11). 总结
> 结合支配树上的信息,以及源码,得出以下结论:    
> 1. IdentityHashMap属于ParserConfig(单例)的成员变量.    
> 2. IdentityHashMap为什么那么多Entry实例(1187个实例),总共占据了:3.41M.  
> 3. 造成OOM的原因是:  
> <font color='red'>FastJson遇到到Type,时会创建一个:JavaObjectDeserializer,并通过Map保存它俩的关系,FastJson希望,下次遇到这个Type时,可以通过Map找到:JavaObjectDeserializer,结果,Map是在不断的增长.既然,Map有类似于Cache的功能,为什么不设计成(SoftReference/WeakReference).</font>   