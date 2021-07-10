---
layout: post
title: 'Spring Cache源码之自定义CacheManager(四)'
date: 2021-07-10
author: 李新
tags:  Spring 
---

### (1). 前言
在前面说过,Spring Cache与Redis结合时,key的过期时间(TTL)是统一的,如果想要做到某个key,有自己的过期时间,就只能自己去扩展源码了.

### (2). 实现方式
> 1. 最简例方法(@Cacheable(cacheNames = "users#PT60s")).    
> 2. 自定义: 注解(@Cacheable(cacheNames = "users", ttl = 60000)),注解解析器CacheOperationSource,注解解析后的数据载体CacheOperation.       
> 我这里使用最简单的,即方案一.方案二需要一天以上的时间来做开发.    

### (3). 自定义CacheManager
> 直接把RedisCacheManager拷贝过来,重命名,注意:package名称不能动,因为,在这个类的内部使用到了其它的包级别的类(注意:我标星的部位).   

```
package org.springframework.data.redis.cache;

import java.time.Duration;
import java.util.Collection;
import java.util.Collections;
import java.util.HashMap;
import java.util.LinkedHashMap;
import java.util.LinkedList;
import java.util.List;
import java.util.Map;
import java.util.Set;

import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.transaction.AbstractTransactionSupportingCacheManager;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.lang.Nullable;
import org.springframework.util.Assert;
import org.springframework.util.StringUtils;

public class RedisCacheNewManager extends AbstractTransactionSupportingCacheManager {

	private final RedisCacheWriter cacheWriter;
	private final RedisCacheConfiguration defaultCacheConfig;
	private final Map<String, RedisCacheConfiguration> initialCacheConfiguration;
	private final boolean allowInFlightCacheCreation;

	/**
	 * Creates new {@link RedisCacheManager} using given {@link RedisCacheWriter}
	 * and default {@link RedisCacheConfiguration}.
	 *
	 * @param cacheWriter                must not be {@literal null}.
	 * @param defaultCacheConfiguration  must not be {@literal null}. Maybe just use
	 *                                   {@link RedisCacheConfiguration#defaultCacheConfig()}.
	 * @param allowInFlightCacheCreation allow create unconfigured caches.
	 * @since 2.0.4
	 */
	private RedisCacheNewManager(RedisCacheWriter cacheWriter, RedisCacheConfiguration defaultCacheConfiguration,
			boolean allowInFlightCacheCreation) {

		Assert.notNull(cacheWriter, "CacheWriter must not be null!");
		Assert.notNull(defaultCacheConfiguration, "DefaultCacheConfiguration must not be null!");

		this.cacheWriter = cacheWriter;
		this.defaultCacheConfig = defaultCacheConfiguration;
		this.initialCacheConfiguration = new LinkedHashMap<>();
		this.allowInFlightCacheCreation = allowInFlightCacheCreation;
	}

	/**
	 * Creates new {@link RedisCacheManager} using given {@link RedisCacheWriter}
	 * and default {@link RedisCacheConfiguration}.
	 *
	 * @param cacheWriter               must not be {@literal null}.
	 * @param defaultCacheConfiguration must not be {@literal null}. Maybe just use
	 *                                  {@link RedisCacheConfiguration#defaultCacheConfig()}.
	 */
	public RedisCacheNewManager(RedisCacheWriter cacheWriter, RedisCacheConfiguration defaultCacheConfiguration) {
		this(cacheWriter, defaultCacheConfiguration, true);
	}

	/**
	 * Creates new {@link RedisCacheManager} using given {@link RedisCacheWriter}
	 * and default {@link RedisCacheConfiguration}.
	 *
	 * @param cacheWriter               must not be {@literal null}.
	 * @param defaultCacheConfiguration must not be {@literal null}. Maybe just use
	 *                                  {@link RedisCacheConfiguration#defaultCacheConfig()}.
	 * @param initialCacheNames         optional set of known cache names that will
	 *                                  be created with given
	 *                                  {@literal defaultCacheConfiguration}.
	 */
	public RedisCacheNewManager(RedisCacheWriter cacheWriter, RedisCacheConfiguration defaultCacheConfiguration,
			String... initialCacheNames) {

		this(cacheWriter, defaultCacheConfiguration, true, initialCacheNames);
	}

	/**
	 * Creates new {@link RedisCacheManager} using given {@link RedisCacheWriter}
	 * and default {@link RedisCacheConfiguration}.
	 *
	 * @param cacheWriter                must not be {@literal null}.
	 * @param defaultCacheConfiguration  must not be {@literal null}. Maybe just use
	 *                                   {@link RedisCacheConfiguration#defaultCacheConfig()}.
	 * @param allowInFlightCacheCreation if set to {@literal true} no new caches can
	 *                                   be acquire at runtime but limited to the
	 *                                   given list of initial cache names.
	 * @param initialCacheNames          optional set of known cache names that will
	 *                                   be created with given
	 *                                   {@literal defaultCacheConfiguration}.
	 * @since 2.0.4
	 */
	public RedisCacheNewManager(RedisCacheWriter cacheWriter, RedisCacheConfiguration defaultCacheConfiguration,
			boolean allowInFlightCacheCreation, String... initialCacheNames) {

		this(cacheWriter, defaultCacheConfiguration, allowInFlightCacheCreation);

		for (String cacheName : initialCacheNames) {
			this.initialCacheConfiguration.put(cacheName, defaultCacheConfiguration);
		}
	}

	/**
	 * Creates new {@link RedisCacheManager} using given {@link RedisCacheWriter}
	 * and default {@link RedisCacheConfiguration}.
	 *
	 * @param cacheWriter                must not be {@literal null}.
	 * @param defaultCacheConfiguration  must not be {@literal null}. Maybe just use
	 *                                   {@link RedisCacheConfiguration#defaultCacheConfig()}.
	 * @param initialCacheConfigurations Map of known cache names along with the
	 *                                   configuration to use for those caches. Must
	 *                                   not be {@literal null}.
	 */
	public RedisCacheNewManager(RedisCacheWriter cacheWriter, RedisCacheConfiguration defaultCacheConfiguration,
			Map<String, RedisCacheConfiguration> initialCacheConfigurations) {

		this(cacheWriter, defaultCacheConfiguration, initialCacheConfigurations, true);
	}

	/**
	 * Creates new {@link RedisCacheManager} using given {@link RedisCacheWriter}
	 * and default {@link RedisCacheConfiguration}.
	 *
	 * @param cacheWriter                must not be {@literal null}.
	 * @param defaultCacheConfiguration  must not be {@literal null}. Maybe just use
	 *                                   {@link RedisCacheConfiguration#defaultCacheConfig()}.
	 * @param initialCacheConfigurations Map of known cache names along with the
	 *                                   configuration to use for those caches. Must
	 *                                   not be {@literal null}.
	 * @param allowInFlightCacheCreation if set to {@literal false} this cache
	 *                                   manager is limited to the initial cache
	 *                                   configurations and will not create new
	 *                                   caches at runtime.
	 * @since 2.0.4
	 */
	public RedisCacheNewManager(RedisCacheWriter cacheWriter, RedisCacheConfiguration defaultCacheConfiguration,
			Map<String, RedisCacheConfiguration> initialCacheConfigurations, boolean allowInFlightCacheCreation) {

		this(cacheWriter, defaultCacheConfiguration, allowInFlightCacheCreation);

		Assert.notNull(initialCacheConfigurations, "InitialCacheConfigurations must not be null!");

		this.initialCacheConfiguration.putAll(initialCacheConfigurations);
	}

	/**
	 * Create a new {@link RedisCacheManager} with defaults applied.
	 * <dl>
	 * <dt>locking</dt>
	 * <dd>disabled</dd>
	 * <dt>cache configuration</dt>
	 * <dd>{@link RedisCacheConfiguration#defaultCacheConfig()}</dd>
	 * <dt>initial caches</dt>
	 * <dd>none</dd>
	 * <dt>transaction aware</dt>
	 * <dd>no</dd>
	 * <dt>in-flight cache creation</dt>
	 * <dd>enabled</dd>
	 * </dl>
	 *
	 * @param connectionFactory must not be {@literal null}.
	 * @return new instance of {@link RedisCacheManager}.
	 */
	public static RedisCacheNewManager create(RedisConnectionFactory connectionFactory) {

		Assert.notNull(connectionFactory, "ConnectionFactory must not be null!");

		return new RedisCacheNewManager(new DefaultRedisCacheWriter(connectionFactory),
				RedisCacheConfiguration.defaultCacheConfig());
	}

	/**
	 * Entry point for builder style {@link RedisCacheManager} configuration.
	 *
	 * @param connectionFactory must not be {@literal null}.
	 * @return new {@link RedisCacheManagerBuilder}.
	 */
	public static RedisCacheManagerBuilder builder(RedisConnectionFactory connectionFactory) {

		Assert.notNull(connectionFactory, "ConnectionFactory must not be null!");

		return RedisCacheManagerBuilder.fromConnectionFactory(connectionFactory);
	}

	/**
	 * Entry point for builder style {@link RedisCacheManager} configuration.
	 *
	 * @param cacheWriter must not be {@literal null}.
	 * @return new {@link RedisCacheManagerBuilder}.
	 */
	public static RedisCacheManagerBuilder builder(RedisCacheWriter cacheWriter) {
		Assert.notNull(cacheWriter, "CacheWriter must not be null!");
		return RedisCacheManagerBuilder.fromCacheWriter(cacheWriter);
	}

	/*
	 * (non-Javadoc)
	 * 
	 * @see org.springframework.cache.support.AbstractCacheManager#loadCaches()
	 */
	@Override
	protected Collection<RedisCache> loadCaches() {
		List<RedisCache> caches = new LinkedList<>();
		for (Map.Entry<String, RedisCacheConfiguration> entry : initialCacheConfiguration.entrySet()) {
			caches.add(createRedisCache(entry.getKey(), entry.getValue()));
		}
		return caches;
	}

	/*
	 * (non-Javadoc)
	 * 
	 * @see
	 * org.springframework.cache.support.AbstractCacheManager#getMissingCache(java.
	 * lang.String)
	 */
	@Override
	protected RedisCache getMissingCache(String name) {
		return allowInFlightCacheCreation ? createRedisCache(name, defaultCacheConfig) : null;
	}

	/**
	 * @return unmodifiable {@link Map} containing cache name / configuration pairs.
	 *         Never {@literal null}.
	 */
	public Map<String, RedisCacheConfiguration> getCacheConfigurations() {

		Map<String, RedisCacheConfiguration> configurationMap = new HashMap<>(getCacheNames().size());

		getCacheNames().forEach(it -> {
			RedisCache cache = RedisCache.class.cast(lookupCache(it));
			configurationMap.put(it, cache != null ? cache.getCacheConfiguration() : null);
		});

		return Collections.unmodifiableMap(configurationMap);
	}

	/**
	 * Configuration hook for creating {@link RedisCache} with given name and
	 * {@code cacheConfig}.
	 *
	 * @param name        must not be {@literal null}.
	 * @param cacheConfig can be {@literal null}.
	 * @return never {@literal null}.
	 */
	protected RedisCache createRedisCache(String name, @Nullable RedisCacheConfiguration cacheConfig) {
		// ********************************************************************************
		// 1. 判断名字是否包含有指定字符(#)
		// ********************************************************************************
		RedisCacheConfiguration tmpCacheConfig = cacheConfig != null ? cacheConfig : defaultCacheConfig;
		String tmpName = name;
		int position = name.indexOf("#");
		if (position > 0) {
			// 缓存名称
			tmpName = name.substring(0, position);
			// 过期时间
			String ttl = name.substring(position + 1);
			// 重新设置过期时间(内部实际是重新new出了一个RedisCacheConfiguration对象)
			tmpCacheConfig = tmpCacheConfig.entryTtl(Duration.parse(ttl));
		}
		
		return new RedisCache(tmpName, cacheWriter, tmpCacheConfig);
	}

	/**
	 * Configurator for creating {@link RedisCacheManager}.
	 *
	 * @author Christoph Strobl
	 * @author Mark Strobl
	 * @author Kezhu Wang
	 * @since 2.0
	 */
	public static class RedisCacheManagerBuilder {

		private final RedisCacheWriter cacheWriter;
		private RedisCacheConfiguration defaultCacheConfiguration = RedisCacheConfiguration.defaultCacheConfig();
		private final Map<String, RedisCacheConfiguration> initialCaches = new LinkedHashMap<>();
		private boolean enableTransactions;
		boolean allowInFlightCacheCreation = true;

		private RedisCacheManagerBuilder(RedisCacheWriter cacheWriter) {
			this.cacheWriter = cacheWriter;
		}

		/**
		 * Entry point for builder style {@link RedisCacheManager} configuration.
		 *
		 * @param connectionFactory must not be {@literal null}.
		 * @return new {@link RedisCacheManagerBuilder}.
		 */
		public static RedisCacheManagerBuilder fromConnectionFactory(RedisConnectionFactory connectionFactory) {

			Assert.notNull(connectionFactory, "ConnectionFactory must not be null!");

			return builder(new DefaultRedisCacheWriter(connectionFactory));
		}

		/**
		 * Entry point for builder style {@link RedisCacheManager} configuration.
		 *
		 * @param cacheWriter must not be {@literal null}.
		 * @return new {@link RedisCacheManagerBuilder}.
		 */
		public static RedisCacheManagerBuilder fromCacheWriter(RedisCacheWriter cacheWriter) {

			Assert.notNull(cacheWriter, "CacheWriter must not be null!");

			return new RedisCacheManagerBuilder(cacheWriter);
		}

		/**
		 * Define a default {@link RedisCacheConfiguration} applied to dynamically
		 * created {@link RedisCache}s.
		 *
		 * @param defaultCacheConfiguration must not be {@literal null}.
		 * @return this {@link RedisCacheManagerBuilder}.
		 */
		public RedisCacheManagerBuilder cacheDefaults(RedisCacheConfiguration defaultCacheConfiguration) {

			Assert.notNull(defaultCacheConfiguration, "DefaultCacheConfiguration must not be null!");

			this.defaultCacheConfiguration = defaultCacheConfiguration;

			return this;
		}

		/**
		 * Enable {@link RedisCache}s to synchronize cache put/evict operations with
		 * ongoing Spring-managed transactions.
		 *
		 * @return this {@link RedisCacheManagerBuilder}.
		 */
		public RedisCacheManagerBuilder transactionAware() {

			this.enableTransactions = true;

			return this;
		}

		/**
		 * Append a {@link Set} of cache names to be pre initialized with current
		 * {@link RedisCacheConfiguration}. <strong>NOTE:</strong> This calls depends on
		 * {@link #cacheDefaults(RedisCacheConfiguration)} using whatever default
		 * {@link RedisCacheConfiguration} is present at the time of invoking this
		 * method.
		 *
		 * @param cacheNames must not be {@literal null}.
		 * @return this {@link RedisCacheManagerBuilder}.
		 */
		public RedisCacheManagerBuilder initialCacheNames(Set<String> cacheNames) {

			Assert.notNull(cacheNames, "CacheNames must not be null!");

			Map<String, RedisCacheConfiguration> cacheConfigMap = new LinkedHashMap<>(cacheNames.size());
			cacheNames.forEach(it -> cacheConfigMap.put(it, defaultCacheConfiguration));

			return withInitialCacheConfigurations(cacheConfigMap);
		}

		/**
		 * Append a {@link Map} of cache name/{@link RedisCacheConfiguration} pairs to
		 * be pre initialized.
		 *
		 * @param cacheConfigurations must not be {@literal null}.
		 * @return this {@link RedisCacheManagerBuilder}.
		 */
		public RedisCacheManagerBuilder withInitialCacheConfigurations(
				Map<String, RedisCacheConfiguration> cacheConfigurations) {

			Assert.notNull(cacheConfigurations, "CacheConfigurations must not be null!");
			cacheConfigurations.forEach((cacheName, configuration) -> Assert.notNull(configuration,
					String.format("RedisCacheConfiguration for cache %s must not be null!", cacheName)));

			this.initialCaches.putAll(cacheConfigurations);

			return this;
		}

		/**
		 * Disable in-flight {@link org.springframework.cache.Cache} creation for
		 * unconfigured caches.
		 * <p />
		 * {@link RedisCacheManager#getMissingCache(String)} returns {@literal null} for
		 * any unconfigured {@link org.springframework.cache.Cache} instead of a new
		 * {@link RedisCache} instance. This allows eg.
		 * {@link org.springframework.cache.support.CompositeCacheManager} to chime in.
		 *
		 * @return this {@link RedisCacheManagerBuilder}.
		 * @since 2.0.4
		 */
		public RedisCacheManagerBuilder disableCreateOnMissingCache() {

			this.allowInFlightCacheCreation = false;
			return this;
		}

		/**
		 * Create new instance of {@link RedisCacheManager} with configuration options
		 * applied.
		 *
		 * @return new instance of {@link RedisCacheManager}.
		 */
		// ********************************************************************************
		// 2. 通过Builder对象,返回的是一个新的:RedisCacheNewManager
		// ********************************************************************************
		public RedisCacheNewManager build() {
			RedisCacheNewManager cm = new RedisCacheNewManager(cacheWriter, defaultCacheConfiguration, initialCaches,
					allowInFlightCacheCreation);
			cm.setTransactionAware(enableTransactions);
			return cm;
		}
	}
}
```
### (4). 配置CacheManager
```
package help.lixin.config;

import java.util.LinkedHashSet;
import java.util.List;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.cache.CacheProperties;
import org.springframework.boot.autoconfigure.cache.CacheProperties.Redis;
import org.springframework.boot.autoconfigure.data.redis.RedisProperties;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.cache.CacheManager;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.ResourceLoader;
import org.springframework.data.redis.cache.RedisCacheNewManager;
import org.springframework.data.redis.cache.RedisCacheNewManager.RedisCacheManagerBuilder;
import org.springframework.data.redis.connection.RedisClusterConfiguration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.connection.RedisPassword;
import org.springframework.data.redis.connection.jedis.JedisConnectionFactory;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializationContext.SerializationPair;

import redis.clients.jedis.JedisPoolConfig;

@Configuration
@EnableConfigurationProperties(CacheProperties.class)
public class RedisConnectionFactoryConfig {

	@Autowired
	private RedisProperties redisProperties;

	@Autowired
	private CacheProperties cacheProperties;

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
	
	
	// ********************************************
	// 1. 使用自定义的RedisCacheNewManager
	// ********************************************
	@Bean
	public CacheManager cacheManager(RedisConnectionFactory redisConnectionFactory,
			ResourceLoader resourceLoader) {
		RedisCacheManagerBuilder builder = RedisCacheNewManager
				.builder(redisConnectionFactory)
				.cacheDefaults(determineConfiguration(resourceLoader.getClassLoader()));
		List<String> cacheNames = this.cacheProperties.getCacheNames();
		if (!cacheNames.isEmpty()) {
			builder.initialCacheNames(new LinkedHashSet<>(cacheNames));
		}
		return builder.build();
	}

	private org.springframework.data.redis.cache.RedisCacheConfiguration determineConfiguration(
			ClassLoader classLoader) {
		Redis redisProperties = this.cacheProperties.getRedis();
		org.springframework.data.redis.cache.RedisCacheConfiguration config = org.springframework.data.redis.cache.RedisCacheConfiguration
				.defaultCacheConfig();
		// json序列化
		config = config.serializeValuesWith(SerializationPair.fromSerializer(RedisSerializer.json()));
		// # spring.cache.redis.time-to-live=PT60s
		// 在配置文件中,指定的缓存过期时间,代表的是所有KEY的过期时间.
		if (redisProperties.getTimeToLive() != null) {
			config = config.entryTtl(redisProperties.getTimeToLive());
		}
		if (redisProperties.getKeyPrefix() != null) {
			config = config.prefixKeysWith(redisProperties.getKeyPrefix());
		}
		if (!redisProperties.isCacheNullValues()) {
			config = config.disableCachingNullValues();
		}
		if (!redisProperties.isUseKeyPrefix()) {
			config = config.disableKeyPrefix();
		}
		return config;
	}
}
```
### (5). 测试验证
```
127.0.0.1:6382> keys *
1) "users::1"

127.0.0.1:6382> GET "users::1"
"[\"java.util.ArrayList\",[{\"@class\":\"help.lixin.entity.User\",\"id\":1,\"name\":\"\xe5\xbc\xa0\xe4\xb8\x89\"},{\"@class\":\"help.lixin.entity.User\",\"id\":2,\"name\":\"\xe6\x9d\x8e\xe5\x9b\x9b\"},{\"@class\":\"help.lixin.entity.User\",\"id\":3,\"name\":\"\xe7\x8e\x8b\xe4\xba\x94\"},{\"@class\":\"help.lixin.entity.User\",\"id\":4,\"name\":\"\xe8\xb5\xb5\xe5\x85\xad\"}]]"

# 查看TTL过期时间
127.0.0.1:6382> TTL "users::1"
(integer) 41
```
### (6). 总结
通过简单的改造,就可以实现针对某一key配置过期时间.实际,像缓存穿透(布隆过滤器),还是需要进一步对:CacheInterceptor进行扩展的.    