---
layout: post
title: '分布式锁之 LockRegistry'
date: 2021-07-15
author: 李新
tags:  Redis Spring
---

### (1). 概述
在前面源码分析过Redssion,昨晚,睡觉时刷到:RedisLockRegistry,比较好奇,起床第一件事就是尝试一把,结果出乎我的意料,总结下缺点:      
1) Redis不能是集群模式,否则,会抛错(Redssion不会有这情况).              
2) LockRegistry为key配置的过期时间默认为60s,如果业务执行80秒,在key到达60秒时会消失,LockRegistry并不会自动续锁,有点恐怖哈.         

### (2). 添加依赖
```
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

<!-- redis-lock -->
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-integration</artifactId>
</dependency>
<dependency>
   <groupId>org.springframework.integration</groupId>
   <artifactId>spring-integration-redis</artifactId>
</dependency>

<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-test</artifactId>
	<scope>test</scope>
</dependency>
```
### (3). RedisConnectionFactoryConfig
```
// 1. 配置锁
@Bean
public LockRegistry lockRegistry(RedisConnectionFactory redisConnectionFactory) {
	// test          : 为key的前缀
	// expireAfter   : 过期时间(注意:默认是60秒)
	// 问题来了,如果我业务执行有80秒,会不会帮我自动续锁?
	RedisLockRegistry redisLockRegistry = new RedisLockRegistry(redisConnectionFactory, "test");
	return redisLockRegistry;
}

// 2. 配置连接池工厂
@Bean
public RedisConnectionFactory redisConnectionFactory() {
	JedisShardInfo jedisShardInfo = new JedisShardInfo("127.0.0.1", 6379);
	return new JedisConnectionFactory(jedisShardInfo);
}
```
### (4). HelloService
```
@Service
public class HelloService implements IHelloService {
	private Logger logger = LoggerFactory.getLogger(HelloService.class);

	@Autowired
	private LockRegistry lockRegistry;

	public List<User> findByTenantId(Long tenantId) {
		List<User> results = new ArrayList<User>();
		
		Lock lock = lockRegistry.obtain("users");
		try {
			// 加锁
			lock.lock();
			
			System.out.println("lock start");
			// 休息80秒,看Redis中KEY的状态.
			TimeUnit.SECONDS.sleep(80);
			// ... ...
			System.out.println("lock end");
		} catch (InterruptedException e) {
			e.printStackTrace();
		} finally {
			// 释放锁
			lock.unlock();
		}
		return results;
	} // end findByTenantId
}// end 
```
### (5). 测试结果
```
127.0.0.1:6379> keys *
1) "test:users"
127.0.0.1:6379> TTL "test:users"
(integer) 49
127.0.0.1:6379> TTL "test:users"
(integer) 47

// 60秒后....

127.0.0.1:6379> TTL "test:users"
(integer) 0
127.0.0.1:6379> TTL "test:users"
(integer) -2
127.0.0.1:6379> keys *
(empty list or set)
```
### (6). 缺陷
```
# 在Redis集群模式下,LockRegistry报错如下:
# 集群模式下不支持evalSha函数.
org.springframework.dao.InvalidDataAccessApiUsageException: EvalSha is not supported in cluster environment.
```