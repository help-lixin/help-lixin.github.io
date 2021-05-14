---
layout: post
title: 'Spring Cloud Gateway RedisRateLimiter(十)'
date: 2020-12-16
author: 李新
tags: SpringCloudGateway源码 令牌桶算法限流
---

### (1). 概述
> SpringCloudGateway针对分布式限流,进行了抽象(RateLimiter)和实现:RedisRateLimiter.  
> 在这里,主要剖析:SpringCloudGateway是如何利用:Redis+Lua实现分布式限流的.  

### (2). application.yml
```
#端口
server:
  port: 9000

spring:
  application:
    name: gateway-server  # 应用名称
  
  redis:
    timeout: 10000
    host: 127.0.0.1
    port: 6379 
    database: 0
    lettuce:
        pool:
          max-active: 1024 #连接池最大连接数（负值表示没有限制）
          max-wait: 10000 #连接池最大阻塞等待时间（负值表示没有限制）
          max-idle: 200 #连接池最大空闭连接数
          min-idle: 5 #连接汉最小空闲连接数
  
  cloud:
    gateway:
      routes:
        - id: test-consumer-service
          uri: "http://localhost:7070/"
          predicates: 
            - Path=/consumer/**
          filters:
            - name: RequestRateLimiter
              args:
                  redis-rate-limiter.replenishRate: 1    # 令牌桶每秒填充速率(代表:每秒生成一个令牌)
                  redis-rate-limiter.burstCapacity: 2    # 令版桶总容量
                  redis-rate-limiter.requestedTokens: 1  # 每次请求消费一个令牌
                  key-resolver: "#{@pathKeyResolver}"     # SpElL表达式按名称引用bean 
```
### (3). pom.xml
```
<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<maven.compiler.source>1.8</maven.compiler.source>
		<maven.compiler.target>1.8</maven.compiler.target>
		<java.version>1.8</java.version>
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
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-gateway</artifactId>
		</dependency>
		<!-- 引入:redis-reactive -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-redis-reactive</artifactId>
		</dependency>
		<dependency>
			<groupId>org.apache.commons</groupId>
			<artifactId>commons-pool2</artifactId>
		</dependency>
	</dependencies>
</project>
```
### (4). RedisRateLimiter(限流源码入口)
```
// 是否允许请求通过.
public Mono<Response> isAllowed(
               // test-consumer-service
               String routeId, 
			   // /consumer(url请求)
			   String id) {
    // 获得配置信息
	Config routeConfig = loadConfiguration(routeId);
	
	// How many requests per second do you want a user to be allowed to do?
	// redis-rate-limiter.replenishRate: 1    # 令牌桶每秒填充速率(代表:每秒生成一个令牌)
	int replenishRate = routeConfig.getReplenishRate();

	// How much bursting do you want to allow?
	// redis-rate-limiter.burstCapacity: 2    # 令版桶总容量
	int burstCapacity = routeConfig.getBurstCapacity();
	
	try {
		// 生成Redis中的唯一Key
		List<String> keys = getKeys(id);

        //  *****************************************************************
		// 这个参数:requestedTokens,直接被写死了,变成了:1,是有什么梗吗?
		//  *****************************************************************
		// The arguments to the LUA script. time() returns unixtime in seconds.
		List<String> scriptArgs = Arrays.asList(replenishRate + "", burstCapacity + "",
				Instant.now().getEpochSecond() + "", "1");
		// allowed, tokens_left = redis.eval(SCRIPT, keys, args)
		
		// 运行redis(lua)脚本.
		Flux<List<Long>> flux = this.redisTemplate.execute(this.script, keys, scriptArgs);
				// .log("redisratelimiter", Level.FINER);
		return flux.onErrorResume(throwable -> Flux.just(Arrays.asList(1L, -1L)))
				.reduce(new ArrayList<Long>(), (longs, l) -> {
					longs.addAll(l);
					return longs;
				}) .map(results -> {
					boolean allowed = results.get(0) == 1L;
					Long tokensLeft = results.get(1);

					Response response = new Response(allowed, getHeaders(routeConfig, tokensLeft));

					if (log.isDebugEnabled()) {
						log.debug("response: " + response);
					}
					return response;
				});
	} catch (Exception e) {
		log.error("Error determining if user allowed from redis", e);
	}
	// 
	return Mono.just(new Response(true, getHeaders(routeConfig, -1L)));
} // end isAllowed

static List<String> getKeys(String id) {
	// Make a unique key per user.
	String prefix = "request_rate_limiter.{" + id;

	// You need two Redis keys for Token Bucket.
	String tokenKey = prefix + "}.tokens";
	String timestampKey = prefix + "}.timestamp";
	return Arrays.asList(tokenKey, timestampKey);
} // end getKeys
```
### (5). request_rate_limiter.lua脚本
```
# KEYS = [request_rate_limiter.{/consumer}.tokens, request_rate_limiter.{/consumer}.timestamp]
# ARGVS = [1, 2, 1620926324, 1]

local tokens_key = KEYS[1]
local timestamp_key = KEYS[2]
--redis.log(redis.LOG_WARNING, "tokens_key " .. tokens_key)

local rate = tonumber(ARGV[1])
local capacity = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])

local fill_time = capacity/rate
local ttl = math.floor(fill_time*2)

--redis.log(redis.LOG_WARNING, "rate " .. ARGV[1])
--redis.log(redis.LOG_WARNING, "capacity " .. ARGV[2])
--redis.log(redis.LOG_WARNING, "now " .. ARGV[3])
--redis.log(redis.LOG_WARNING, "requested " .. ARGV[4])
--redis.log(redis.LOG_WARNING, "filltime " .. fill_time)
--redis.log(redis.LOG_WARNING, "ttl " .. ttl)

local last_tokens = tonumber(redis.call("get", tokens_key))
if last_tokens == nil then
  last_tokens = capacity
end
--redis.log(redis.LOG_WARNING, "last_tokens " .. last_tokens)

local last_refreshed = tonumber(redis.call("get", timestamp_key))
if last_refreshed == nil then
  last_refreshed = 0
end
--redis.log(redis.LOG_WARNING, "last_refreshed " .. last_refreshed)

local delta = math.max(0, now-last_refreshed)
local filled_tokens = math.min(capacity, last_tokens+(delta*rate))
local allowed = filled_tokens >= requested
local new_tokens = filled_tokens
local allowed_num = 0
if allowed then
  new_tokens = filled_tokens - requested
  allowed_num = 1
end

--redis.log(redis.LOG_WARNING, "delta " .. delta)
--redis.log(redis.LOG_WARNING, "filled_tokens " .. filled_tokens)
--redis.log(redis.LOG_WARNING, "allowed_num " .. allowed_num)
--redis.log(redis.LOG_WARNING, "new_tokens " .. new_tokens)

if ttl > 0 then
  redis.call("setex", tokens_key, ttl, new_tokens)
  redis.call("setex", timestamp_key, ttl, now)
end

-- return { allowed_num, new_tokens, capacity, filled_tokens, requested, new_tokens }
return { allowed_num, new_tokens }
```

### (6). 日志验证
```
# 需求:令牌桶的总数为:2个,每秒产生1个令牌,每秒只允许消费1个令牌.

# 在Redis中定义要限流的资源('request_rate_limiter.{/consumer}.tokens')
3172:M 14 May 2021 12:09:19.410 # tokens_key request_rate_limiter.{/consumer}.tokens
# 令牌桶每秒填充速率(代表:每秒生成一个令牌)
3172:M 14 May 2021 12:09:19.411 # rate 1
# 令牌桶的总容量
3172:M 14 May 2021 12:09:19.411 # capacity 2
# 当前时间
3172:M 14 May 2021 12:09:19.411 # now 1620965359
# 令牌桶每秒消费数量
3172:M 14 May 2021 12:09:19.411 # requested 1

# 填充时间
3172:M 14 May 2021 12:09:19.412 # filltime 2
3172:M 14 May 2021 12:09:19.412 # ttl 4

3172:M 14 May 2021 12:09:19.412 # last_tokens 2
3172:M 14 May 2021 12:09:19.413 # last_refreshed 0
3172:M 14 May 2021 12:09:19.413 # delta 1620965359
3172:M 14 May 2021 12:09:19.413 # filled_tokens 2
3172:M 14 May 2021 12:09:19.413 # allowed_num 1
3172:M 14 May 2021 12:09:19.414 # new_tokens 1

3172:M 14 May 2021 12:09:19.414 # tokens_key request_rate_limiter.{/consumer}.tokens
3172:M 14 May 2021 12:09:19.415 # rate 1
3172:M 14 May 2021 12:09:19.415 # capacity 2
3172:M 14 May 2021 12:09:19.415 # now 1620965359
3172:M 14 May 2021 12:09:19.416 # requested 1
3172:M 14 May 2021 12:09:19.416 # filltime 2
3172:M 14 May 2021 12:09:19.416 # ttl 4
3172:M 14 May 2021 12:09:19.417 # last_tokens 1
3172:M 14 May 2021 12:09:19.417 # last_refreshed 1620965359
3172:M 14 May 2021 12:09:19.418 # delta 0
3172:M 14 May 2021 12:09:19.418 # filled_tokens 1
3172:M 14 May 2021 12:09:19.418 # allowed_num 1
3172:M 14 May 2021 12:09:19.419 # new_tokens 0

3172:M 14 May 2021 12:09:19.419 # tokens_key request_rate_limiter.{/consumer}.tokens
3172:M 14 May 2021 12:09:19.419 # rate 1
3172:M 14 May 2021 12:09:19.420 # capacity 2
3172:M 14 May 2021 12:09:19.421 # now 1620965359
3172:M 14 May 2021 12:09:19.422 # requested 1
3172:M 14 May 2021 12:09:19.423 # filltime 2
3172:M 14 May 2021 12:09:19.424 # ttl 4
3172:M 14 May 2021 12:09:19.424 # last_tokens 0
3172:M 14 May 2021 12:09:19.425 # last_refreshed 1620965359
3172:M 14 May 2021 12:09:19.426 # delta 0
3172:M 14 May 2021 12:09:19.426 # filled_tokens 0
3172:M 14 May 2021 12:09:19.427 # allowed_num 0
3172:M 14 May 2021 12:09:19.427 # new_tokens 0
```
### (7). 总结
