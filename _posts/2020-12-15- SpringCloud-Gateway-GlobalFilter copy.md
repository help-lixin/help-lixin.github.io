---
layout: post
title: 'Spring Cloud Gateway RequestRateLimiter限流(十八)'
date: 2020-12-15
author: 李新
tags: SpringCloudGateway
---

### (1). Filter RequestRateLimiterGatewayFilterFactory
> Spring Cloud Gateway提供了RequestRateLimiter限流,它依赖:Redis和Lua脚本.

### (2). 添加依赖
```
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-data-redis-reactive</artifactId>
</dependency>
<dependency>
	<groupId>org.apache.commons</groupId>
	<artifactId>commons-pool2</artifactId>
</dependency>
```
### (3). application.yml
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
### (4). 配置KeyResolver
```
package help.lixin.conf;

import org.springframework.cloud.gateway.filter.ratelimit.KeyResolver;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import reactor.core.publisher.Mono;

@Configuration
public class KeyResolverConiguration {

	@Bean
	public KeyResolver pathKeyResolver() {
		// ****************************************************
		// URL限流
		// 意味着:可以自定义key(可以把请求变成用户ID)
		// IP限流
		// return (exchange) -> Mono.just(exchange.getRequest().getRemoteAddress().toString());
		// ****************************************************
		return (exchange) -> Mono.just(exchange.getRequest().getPath().toString());
	}
}
```
### (5). 测试
!["限流快速测试"](/assets/spring-cloud-gateway/imgs/spring-cloud-gateway-rate-limiter.png)

### (6). 查看redis信息
```
127.0.0.1:6379> keys *
1) "request_rate_limiter.{/consumer}.timestamp"
2) "request_rate_limiter.{/consumer}.tokens"
127.0.0.1:6379> get "request_rate_limiter.{/consumer}.timestamp"
"1608110283"
127.0.0.1:6379> get "request_rate_limiter.{/consumer}.tokens"
"0"
```
### (7). 总结
> 1. redis-rate-limiter.replenishRate: 1,为令牌桶每秒填充速率(代表每秒向令牌桶添加一个令牌).    
> 2. redis-rate-limiter.burstCapacity: 2,为令版桶总容量.   
> 3. 在RequestRateLimiter的内部,每一次执行请求,都记住最后一次的时间.    
> 4. 从令牌通拿一个令牌,当拿不到的情况下,代表令牌不足,就触发:5,6,7步.    
> 5. 根据当前时间 减 上一次执行时间,计算出相差了多少"秒",比如:相差10秒.    
> 6. 相差的10秒 * redis-rate-limiter.replenishRate = 要产生的令牌个数为:10个.     
> 7. 向令牌桶产生10个令牌(由于受:redis-rate-limiter.burstCapacity限制,只能生成2个),这个过程实际就是更改Redis的值("request_rate_limiter.{/consumer}.tokens"). 
> 8. 从令牌通拿一个令牌,当拿不到的情况下,代表令牌不足,就触发:5,6,7步.    
> 9. 拿令牌的过程就是对redis进行自减("request_rate_limiter.{/consumer}.tokens")
> 10. 为什么用Redis+Lua,就是为了解决Redis操作一致性问题.   
> 11. <font color='red'>缺点:在整个Spring Cloud Gateway进程里,KeyResolver实例只允许有一个,否则,就会抛出异常.</font>