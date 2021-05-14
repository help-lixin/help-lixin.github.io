---
layout: post
title: 'Redisson RedissonRateLimiter(三)'
date: 2021-05-13
author: 李新
tags:  Redisson 令牌桶算法限流
---

### (1). 概述
> 在前面使用了,RRateLimiter进行令牌桶限流的入门,在这里,对RRateLimiter进行深入源码剖析.    
> RedissonRateLimiter是RRateLimiter的唯一实现.   

### (2). RedissonRateLimiter
```
public RFuture<Boolean> trySetRateAsync(RateType type, long rate, long rateInterval, RateIntervalUnit unit) {
	return commandExecutor.evalWriteAsync(getRawName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
			"redis.call('hsetnx', KEYS[1], 'rate', ARGV[1]);"
		  + "redis.call('hsetnx', KEYS[1], 'interval', ARGV[2]);"
		  + "return redis.call('hsetnx', KEYS[1], 'type', ARGV[3]);",
			Collections.singletonList(getRawName()), rate, unit.toMillis(rateInterval), type.ordinal());
}
```
### (3). 配置限流规则(lua)脚本
```
# 业务需求:每隔1秒(1000毫秒),产生2个令牌.

# KEYS[1] = 'myRateLimiter'
# ARGV[1] = 2
# ARGV[2] = 1000
# ARGV[3] = 0

# 设置产生的令牌数
# HSETNX 'myRateLimiter' 'rate' 2
redis.call('hsetnx', KEYS[1], 'rate', ARGV[1]);

# 设置间隔多长时间
# HSETNX 'myRateLimiter' 'interval' 1000
redis.call('hsetnx', KEYS[1], 'interval', ARGV[2]);

# 设置类型.
# HSETNX 'myRateLimiter' 'type' 0
return redis.call('hsetnx', KEYS[1], 'type', ARGV[3]);
```
### (4). acquire(lua)脚本
```
#################################### 以下部份为参数 #################################### 
# KEYS = [myRateLimiter, {myRateLimiter}:value, {myRateLimiter}:value:688ee5a9-0b89-4aba-9d2b-83c81fe1c67c, {myRateLimiter}:permits, {myRateLimiter}:permits:688ee5a9-0b89-4aba-9d2b-83c81fe1c67c]
# KEYS[1] = 'myRateLimiter'
# KEYS[2] = '{myRateLimiter}:value'
# KEYS[3] = '{myRateLimiter}:value:688ee5a9-0b89-4aba-9d2b-83c81fe1c67c'
# KEYS[4] = '{myRateLimiter}:permits'
# KEYS[5] = '{myRateLimiter}:permits:688ee5a9-0b89-4aba-9d2b-83c81fe1c67c'

# ARGVS = [1, 1620920577843, 3246084489035494703]
# ARGV[1] = 1 
# ARGV[2] = 1620920577843
# ARGV[3] = 3246084489035494703

#################################### 以上部份为参数 #################################### 

local rate = redis.call('hget', KEYS[1], 'rate');
local interval = redis.call('hget', KEYS[1], 'interval');
local type = redis.call('hget', KEYS[1], 'type');
assert(rate ~= false and interval ~= false and type ~= false, 'RateLimiter is not initialized')

local valueName = KEYS[2];
local permitsName = KEYS[4];
if type == '1' then 
  valueName = KEYS[3];
  permitsName = KEYS[5];
end;

assert(tonumber(rate) >= tonumber(ARGV[1]), 'Requested permits amount could not exceed defined rate'); 

local currentValue = redis.call('get', valueName); 
if currentValue ~= false then 
  local expiredValues = redis.call('zrangebyscore', permitsName, 0, tonumber(ARGV[2]) - interval); 
  local released = 0; 
  for i, v in ipairs(expiredValues) do 
    local random, permits = struct.unpack('fI', v);
	released = released + permits;
  end; 
  
  if released > 0 then 
    redis.call('zremrangebyscore', permitsName, 0, tonumber(ARGV[2]) - interval); 
	currentValue = tonumber(currentValue) + released; 
	redis.call('set', valueName, currentValue);
  end;
  
  if tonumber(currentValue) < tonumber(ARGV[1]) then 
    local nearest = redis.call('zrangebyscore', permitsName, '(' .. (tonumber(ARGV[2]) - interval), '+inf', 'withscores', 'limit', 0, 1); 
	return tonumber(nearest[2]) - (tonumber(ARGV[2]) - interval);
  else 
    redis.call('zadd', permitsName, ARGV[2], struct.pack('fI', ARGV[3], ARGV[1])); 
	redis.call('decrby', valueName, ARGV[1]); 
    return nil; 
  end; 
  
else 
   redis.call('set', valueName, rate); 
   redis.call('zadd', permitsName, ARGV[2], struct.pack('fI', ARGV[3], ARGV[1])); 
   redis.call('decrby', valueName, ARGV[1]); 
   return nil; 
end;
```
### (5). 总结
> 