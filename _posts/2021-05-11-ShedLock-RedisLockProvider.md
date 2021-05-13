---
layout: post
title: 'ShedLock Redis分布式加锁源码(三)'
date: 2021-05-11
author: 李新
tags:  ShedLock
---

### (1). 前言
> 因为,我比较关注:Reids分布式锁的实现,所以,也只看:RedisLockProvider

### (2). Redis分布式锁集成
```
// 1. 添加依赖配置
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>

<!-- DataSource初始化(DataSourceConfiguration) -->
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
<dependency>
  <groupId>mysql</groupId>
  <artifactId>mysql-connector-java</artifactId>
  <version>5.1.49</version>
</dependency> 

<!-- ShedLock -->
<dependency>
	<groupId>net.javacrumbs.shedlock</groupId>
	<artifactId>shedlock-spring</artifactId>
	<version>4.23.1-SNAPSHOT</version>
</dependency>
<dependency>
	<groupId>net.javacrumbs.shedlock</groupId>
	<artifactId>shedlock-provider-redis-spring</artifactId>
	<!-- <artifactId>shedlock-provider-jdbc-template</artifactId> -->
	<version>4.23.1-SNAPSHOT</version>
</dependency>
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
	<groupId>redis.clients</groupId>
	<artifactId>jedis</artifactId>
</dependency>


// 2. 配置Redis锁
@Configuration
public class ShedLockConfig {
	@Bean
	public RedisConnectionFactory connectionFactory() {
		JedisPoolConfig poolConfig = new JedisPoolConfig();
		// ... ...
		JedisConnectionFactory jedisConnectionFactory = new JedisConnectionFactory(poolConfig);
		jedisConnectionFactory.setHostName("127.0.0.1");
		jedisConnectionFactory.setPort(6379);
		jedisConnectionFactory.setUsePool(Boolean.TRUE);
		return jedisConnectionFactory;
	}
	
	@Bean
	public LockProvider lockProvider(@Autowired RedisConnectionFactory connectionFactory) {
		return new RedisLockProvider(connectionFactory, "test");
	}
}
```
### (3). LockProvider
> ShedLock为了提供分布式锁的多样化,对分布式锁进行抽象定义,我们看一下这个接口:LockProvider. 

```
public interface LockProvider {
	// 1. 返回对象是:Optional
	// 2. LockConfiguration是配置信息(要加锁的名称/锁住时长)
	Optional<SimpleLock> lock(LockConfiguration lockConfiguration);
}
```
### (4). RedisLockProvider
```
public class RedisLockProvider implements LockProvider {
	// 定义默认的:key前缀
    private static final String KEY_PREFIX_DEFAULT = "job-lock";
	// 定义默认的:环境
    private static final String ENV_DEFAULT = "default";

    private final StringRedisTemplate redisTemplate;
	
	// 环境
    private final String environment;
	// key前缀
    private final String keyPrefix;
   
   
   /*
   *  @SchedulerLock(name = "hello-world", lockAtLeastFor = "20000", lockAtMostFor = "30000")
   *  
   */
   @Override
   public Optional<SimpleLock> lock(@NonNull LockConfiguration lockConfiguration) {
	   
	   // *******************************************************************
	   // 1. 构建key(job-lock:test:hello-world)
	   // *******************************************************************
	   String key = buildKey(lockConfiguration.getName());
	   
	   // *******************************************************************
	   // 2. 计算key在Redis中的,出过期时间(expiration)
	   //  lockConfiguration.getLockAtMostUntil() = now() + 30000 
	   //  getExpiration =  lockConfiguration.getLockAtMostUntil() - now()
	   // *******************************************************************
	   Expiration expiration = getExpiration(lockConfiguration.getLockAtMostUntil());
	   
	   // *******************************************************************
	   // 3. 尝试加锁,成功的情况下,返回: Optional<RedisLock>
	   //    SET_IF_ABSENT:
	   //                 如果KEY存在,则,什么都不做,返回:0.
	   //                 如果KEY不存在,则,设置值,返回:1. 
	   // *******************************************************************
	   if (TRUE.equals(tryToSetExpiration(redisTemplate, key, expiration, SET_IF_ABSENT))) {
		   // 4. 加锁成功的情况下,返回:RedisLock
		   return Optional.of(new RedisLock(key, redisTemplate, lockConfiguration));
	   } else {
		   // 5. 加锁失败的情况下,返回:Optional<?>
		   return Optional.empty();
	   }
   } // end lock
   
   
   // *******************************************************************
   // 3.1 对key进行加锁,并配置过期时间,这个过程保证元子性.
   // *******************************************************************
   private static Boolean tryToSetExpiration(StringRedisTemplate template, String key, Expiration expiration, SetOption option) {
	   return template.execute(connection -> {
		   // key = job-lock:test:hello-world
		   byte[] serializedKey = ((RedisSerializer<String>) template.getKeySerializer()).serialize(key);
		   // value = ADDED:2021-05-12T13:31:45.006Z@lixin-macbook.local
		   //         ADDED:now()@hostname
		   byte[] serializedValue = ((RedisSerializer<String>) template.getValueSerializer()).serialize(String.format("ADDED:%s@%s", toIsoString(ClockProvider.now()), getHostname()));
		   // SETNX(元子操作,如果key存在,什么都不做,返回:0,否则,添加内容,并返回:1)
		   return connection.set(serializedKey, serializedValue, expiration, option);
	   }, false);
   }
   
   // 1.1 构建key
   String buildKey(String lockName) {
	   // job-lock:test:hello-world
	   return String.format("%s:%s:%s", keyPrefix, environment, lockName);
   } // end buildKey
   
   // 2.1 until = now() + 30000(即30秒)
  // 2.3  构建:org.springframework.data.redis.core.types.Expiration(过期时间)
   private static Expiration getExpiration(Instant until) {
	   return Expiration.from(getMsUntil(until), TimeUnit.MILLISECONDS);
   }
   
   // 2.2 计算两个时间,相差多少毫秒
   private static long getMsUntil(Instant until) {
	   return Duration.between(ClockProvider.now(), until).toMillis();
   }
}
```
### (5). RedisLock
```
private static final class RedisLock extends AbstractSimpleLock {
	private final String key;
	private final StringRedisTemplate redisTemplate;

	private RedisLock(String key, StringRedisTemplate redisTemplate, LockConfiguration lockConfiguration) {
		super(lockConfiguration);
		this.key = key;
		this.redisTemplate = redisTemplate;
	}

	
    // 1. 前置条件:抢到锁了,才会调用:unlock方法.<br/>
	// 2. unlock逻辑如下:<br/>
	// 3. 调用Redis,成功加锁的时间 + 最小锁住时间(LockAtLeast) = 加锁最小时间 <br/>
	// 4. 如果:当前系统时间 - 加锁最小时间 <= 0,代表着:锁已经达到加锁最小时间,该释放锁了(并没有用:Lua.)  
	// 5. 否则:给key重新设置过期时间.
	@Override
	public void doUnlock() {
		
		// *************************************************************************
		// lockConfiguration.getLockAtLeastUntil() = now() + 20000
		//  注意:now()为成功抢到分布式锁的时间.
		// *************************************************************************
		Expiration keepLockFor = getExpiration(lockConfiguration.getLockAtLeastUntil());
		
		// lock at least until is in the past
		if (keepLockFor.getExpirationTimeInMilliseconds() <= 0) {
			try {
				// ****************************************************
				// 就这样,直接删了锁
				// A,B两个线程尝试给key(lock)加锁: 
				// 正常情况下:   A先拿到线程(假如锁3秒后过期),B这时候是在等待尝试获取锁.   
				// 失败的情况下: A先拿到线程(假如锁3秒后过期),可是A的业务代码执行了10秒.
				//             在第4秒时(A的key被删了,并没有执行释放锁),而,B抢到了这把锁(不必在乎锁多长时间了).
				//             在第10秒时,A执行释放锁,就会把B加的锁给释放掉.
				// ****************************************************
				redisTemplate.delete(key);
			} catch (Exception e) {
				throw new LockException("Can not remove node", e);
			}
		} else {
			// 重新设置key过期时间为:20秒
			tryToSetExpiration(this.redisTemplate, key, keepLockFor, SetOption.SET_IF_PRESENT);
		}
	}
}
```
### (6). 加锁步骤是什么样的(DefaultLockingTaskExecutor)?
```
public class DefaultLockingTaskExecutor implements LockingTaskExecutor {
	
	// 分布式锁提供者
	private final LockProvider lockProvider;
	
	public DefaultLockingTaskExecutor(@NonNull LockProvider lockProvider) {
		this.lockProvider = requireNonNull(lockProvider);
	}

	public <T> TaskResult<T> executeWithLock(
	                 // 业务方法
	                 @NonNull TaskWithResult<T> task, 
					 // 加锁的配置类
					 @NonNull LockConfiguration lockConfig) throws Throwable {
						 
		// 1. 加锁				 
		Optional<SimpleLock> lock = lockProvider.lock(lockConfig);
		// lockName = hello-world
		String lockName = lockConfig.getName();
        
		// 2. 从线程上下文中(LockAssert)判断,是否已经加锁了
		//    如果:已经加过锁了,则调用:业务方法,返回结果回去.
		if (alreadyLockedBy(lockName)) {
			logger.debug("Already locked '{}'", lockName);
			return TaskResult.result(task.call());
		} else if (lock.isPresent()) {   
			// **************************************************************
			// 3. 加锁成功的情況下
			// **************************************************************
			try {
				// 4.当前线程绑定锁名称
				LockAssert.startLock(lockName);
				logger.debug("Locked '{}', lock will be held at most until {}", lockName, lockConfig.getLockAtMostUntil());
				// 5. 调用:业务方法
				return TaskResult.result(task.call());
			} finally {
				// 6. 当前线程解绑锁名称
				LockAssert.endLock();
				// ***********************************************************************
				// 7. 释放锁.
				// ***********************************************************************
				lock.get().unlock();
				
				// ... ...
			}
		} else {
			//  加锁失败的情况下
			logger.debug("Not executing '{}'. It's locked.", lockName);
			return TaskResult.notExecuted();
		}
	} //end executeWithLock
}
```
### (7). 总结
> Redis分布式锁,在释放锁时,有明显地的Bug.老外写代码也这么粗心大意吗?   