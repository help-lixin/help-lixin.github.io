---
layout: post
title: 'Spring Cache + Redis整合(一)'
date: 2021-07-05
author: 李新
tags:  Spring 
---

### (1). Spring Cache是什么
Spring比较喜欢做的一件事情就是:定义规范(抽象),然后,相同类型的产品对规范进行实现(类似于:桥梁模式),可以理解:Spring Cache是Spring针对Cache定义的一套规范.    
比如:你在工作中要用到:Redis/Memcache/Tair/Guava...等等,使用Spring Cache你可以无缝自由切换(组合)这些缓存的实现.     
<font color='red'>底层实则是:对标有注解的类进行AOP拦截(注意:private/final/this call,Spring是无法代理的.)</font>   

### (2). pom.xml
```
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
		<groupId>com.alibaba</groupId>
		<artifactId>fastjson</artifactId>
		<version>1.2.54</version>
	</dependency>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-web</artifactId>
	</dependency>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-data-redis</artifactId>
	</dependency>
	<dependency>
		<groupId>redis.clients</groupId>
		<artifactId>jedis</artifactId>
	</dependency>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-test</artifactId>
		<scope>test</scope>
	</dependency>
</dependencies>
```
### (3). 启用缓存(@EnableCaching)
```
package help.lixin;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cache.annotation.EnableCaching;

@SpringBootApplication
@EnableCaching
public class App {
	public static void main(String[] args) throws Exception {
		SpringApplication.run(App.class, args);
	}
}
```
### (4). 配置(RedisConnectionFactory)
```
package help.lixin.config;

import java.util.List;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.data.redis.RedisProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisClusterConfiguration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.connection.RedisPassword;
import org.springframework.data.redis.connection.jedis.JedisConnectionFactory;

import redis.clients.jedis.JedisPoolConfig;

@Configuration
public class RedisConnectionFactoryConfig {

	@Autowired
	private RedisProperties redisProperties;
	
	//  针对Redis自定义CacheManager的Value序列化机制
	@Bean
	public CacheManagerCustomizer<CacheManager> cacheManagerCustomizer() {
		CacheManagerCustomizer<CacheManager>  customer = (cacheManager)-> {
			if( cacheManager instanceof RedisCacheManager) {
				RedisCacheManager redisCacheManager = (RedisCacheManager)cacheManager;
				try {
					// *********************************************************
					// 由于:RedisCacheManager没有相关的方法可以改变value序列化行为,所以,只能通过反射或者自定义RedisCacheManager创建过程
					// 我选择了反射的方式
					// *********************************************************
					Field defaultCacheConfigField = RedisCacheManager.class.getDeclaredField("defaultCacheConfig");
					// 允许私有属性可以被访问
					defaultCacheConfigField.setAccessible(true);
					
					RedisCacheConfiguration oldDefaultCacheConfig = (RedisCacheConfiguration)defaultCacheConfigField.get(redisCacheManager);
					// 设置为json序列化
					RedisCacheConfiguration newDefaultCacheConfig = oldDefaultCacheConfig.serializeValuesWith(SerializationPair.fromSerializer(RedisSerializer.json()));
					
					// 设置RedisCacheManager属性
					defaultCacheConfigField.set(redisCacheManager, newDefaultCacheConfig);
				} catch (NoSuchFieldException | SecurityException e) {
					e.printStackTrace();
				} catch (IllegalArgumentException e) {
					e.printStackTrace();
				} catch (IllegalAccessException e) {
					e.printStackTrace();
				}
			}
		};
		return customer;
	}

	@Bean
	public RedisConnectionFactory redisConnectionFactory() {
		List<String> clusterNodes = redisProperties.getCluster().getNodes();
		String passwordAsString = redisProperties.getPassword();
		RedisClusterConfiguration redisClusterConfiguration = new RedisClusterConfiguration(clusterNodes);
		redisClusterConfiguration.setPassword(RedisPassword.of(passwordAsString));

		JedisPoolConfig jedisPoolConfig = new JedisPoolConfig();
		// 最大空闲连接数, 默认8个
		jedisPoolConfig.setMaxIdle(100);
		// 最大连接数, 默认8个
		jedisPoolConfig.setMaxTotal(500);
		// 最小空闲连接数, 默认0
		jedisPoolConfig.setMinIdle(0);
		// 获取连接时的最大等待毫秒数(如果设置为阻塞时BlockWhenExhausted),如果超时就抛异常, 小于零:阻塞不确定的时间,
		// 默认-1
		jedisPoolConfig.setMaxWaitMillis(2000); // 设置2秒
		// 对拿到的connection进行validateObject校验
		jedisPoolConfig.setTestOnBorrow(true);
		return new JedisConnectionFactory(redisClusterConfiguration, jedisPoolConfig);
	}
}
```
### (5). HelloService
```
package help.lixin.service.impl;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

import help.lixin.entity.User;
import help.lixin.service.IHelloService;

@Service
public class HelloService implements IHelloService {
	private Logger logger = LoggerFactory.getLogger(HelloService.class);

	private final Map<Long, List<User>> datas = new ConcurrentHashMap<Long, List<User>>();

	public HelloService() {
		init();
	}

	private void init() {
		Long tenantId = 1L;
		List<User> users = new ArrayList<User>();
		users.add(new User(1L, "张三"));
		users.add(new User(2L, "李四"));
		users.add(new User(3L, "王五"));
		users.add(new User(4L, "赵六"));
		datas.put(tenantId, users);
	}

	// 查询全部都入缓存,key=tenantId
	@Cacheable(cacheNames = "users")
	public List<User> findByTenantId(Long tenantId) {
		return datas.get(tenantId);
	}

	// 有添加操作时,仅清除:tenantId % 2 == 0的缓存信息
	@CacheEvict(cacheNames = "users", key="#tenantId" , condition = "#tenantId % 2 == 0", beforeInvocation = true)
	public Boolean add(Long tenantId, User user) {
		List<User> users = new ArrayList<User>();
		users.add(user);
		if (!datas.containsKey(tenantId)) {
			datas.put(tenantId, users);
		} else {
			datas.get(tenantId).addAll(users);
		}
		return true;
	}
}
```
### (6). HelloController
```
package help.lixin.controller;

import java.util.List;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

import help.lixin.entity.User;
import help.lixin.service.IHelloService;

@RestController
public class HelloController {
	private Logger logger = LoggerFactory.getLogger(HelloController.class);

	@Autowired
	private IHelloService helloService;

	@GetMapping("/{tenantId}")
	public List<User> list(@PathVariable("tenantId") Long tenantId) {
		List<User> users = helloService.findByTenantId(tenantId);
		return users;
	}

	@PostMapping("/{tenantId}")
	public String addUser(@PathVariable("tenantId") Long tenantId, @RequestBody User user) {
		helloService.add(tenantId, user);
		return "SUCCESS";
	}
}
```
### (7). application.properties
```
server.port=8080
spring.application.name=test-provider

spring.redis.password=888888
spring.redis.cluster.nodes=127.0.0.1:6380,127.0.0.1:6381,127.0.0.1:6382,127.0.0.1:6383,127.0.0.1:6384,127.0.0.1:6385
spring.redis.cluster.max-redirects=5
```
### (8). 测试查看Redis结果
```
# 1. 发送curl请求
curl http://localhost:8080/1

# 2. 查看redis内容
lixin-macbook:redis-cluster lixin$ ./bin/redis-cli -c -a 888888 -p 6382
# 返回结果不再是JDK的二进制了,而是JSON文本内容.
127.0.0.1:6382> get "users::1"
"[\"java.util.ArrayList\",[{\"@class\":\"help.lixin.entity.User\",\"id\":1,\"name\":\"\xe5\xbc\xa0\xe4\xb8\x89\"},{\"@class\":\"help.lixin.entity.User\",\"id\":2,\"name\":\"\xe6\x9d\x8e\xe5\x9b\x9b\"},{\"@class\":\"help.lixin.entity.User\",\"id\":3,\"name\":\"\xe7\x8e\x8b\xe4\xba\x94\"},{\"@class\":\"help.lixin.entity.User\",\"id\":4,\"name\":\"\xe8\xb5\xb5\xe5\x85\xad\"}]]"
```
### (9). 总结
RedisCacheConfiguration在配置RedisCacheManager时,默认的value序列化是:JdkSerializationRedisSerializer.  
<font color='red'>如果,想定制序列化的value可以是json,则自己实现CacheManagerCustomizer,并托管给Spring即可,想法是挺好的,结果发现:RedisCacheManager没有提供相应的行为可以改造序列化机制,此时,能做的方式只有两种:要么:反射,要么:RedisCacheManager的创建过程自己托管</font>    