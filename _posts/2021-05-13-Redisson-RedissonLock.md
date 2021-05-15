---
layout: post
title: 'Redisson RedissonLock(二)'
date: 2021-05-13
author: 李新
tags:  Redisson
---

### (1). 概述
> 在前面使用了,RLock进行分布式锁的入门,在这里,对RLock进行深入源码剖析.    
> RedissonLock(非公平锁)是RLock的实现之一.   

### (2). RedissonLock

```
<T> RFuture<T> tryLockInnerAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {
	return evalWriteAsync(getRawName(), LongCodec.INSTANCE, command,
			"if (redis.call('exists', KEYS[1]) == 0) then " +
					"redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
					"redis.call('pexpire', KEYS[1], ARGV[1]); " +
					"return nil; " +
					"end; " +
					"if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
					"redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
					"redis.call('pexpire', KEYS[1], ARGV[1]); " +
					"return nil; " +
					"end; " +
					"return redis.call('pttl', KEYS[1]);",
			Collections.singletonList(getRawName()), unit.toMillis(leaseTime), getLockName(threadId));
}
```
### (3). lock(lua)脚本

```
## 注意:Lua脚本的下标是从1开始的
# KEYS[1] = "anyLock"                                  // redis中的key
# ARGV[1] = 10000                                      // 看门狗,续租的时间(ms)
# ARGV[2] = 48575bd6-4e6d-4485-8f61-255063d81d54:47    // 唯一ID:线程ID

# 判断key是否存在(0:不存在  1:存在)
# EXISTS 'anyLock'
if (redis.call('exists', KEYS[1]) == 0) then 
  # 创建Hash数据结构,并为Hash数据结构,设置过期时间
  
  # HINCRBY 'anyLock' '48575bd6-4e6d-4485-8f61-255063d81d54:47' 1
  redis.call('hincrby', KEYS[1], ARGV[2], 1); 
  
  # PEXPIRE 'anyLock' 10000
  redis.call('pexpire', KEYS[1], ARGV[1]); 
  return nil; 
end; 

# 在Hash表中判断field和value是否存在(0:不存在  1:存在)
# HEXISTS 'anyLock' '48575bd6-4e6d-4485-8f61-255063d81d54:47'
if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then 
  # key在Hash中存在的情况下.
  
  # 让Hash中的key(anyLock),field('48575bd6-4e6d-4485-8f61-255063d81d54:47')进行自增.
  # HINCRBY 'anyLock' '48575bd6-4e6d-4485-8f61-255063d81d54:47' 1
  redis.call('hincrby', KEYS[1], ARGV[2], 1); 
  
  # 重新设置过期时间
  redis.call('pexpire', KEYS[1], ARGV[1]); 
  return nil; 
end; 

# 以毫秒为单位,返回:key的过期时间.
# PTTL 'anyLock'
return redis.call('pttl', KEYS[1]);
```
### (4). unlock(lua)脚本
```

KEYS[1] = 'anyLock' 
KEYS[2] = redisson_lock__channel:{anyLock}]

ARGV[1] = 0
ARGV[2] = 10000
ARGV[3] = 'afc990dd-79b6-4d48-b07e-d900796d7691:48'


# 1. 判断锁是否存在(0:不存在  1:存在)
# HEXISTS 'anyLock' 'afc990dd-79b6-4d48-b07e-d900796d7691:48' 
if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then 
  return nil;
end; 

# 对Hash表中的key(anyLock),field(afc990dd-79b6-4d48-b07e-d900796d7691:48),实现自减.
# HINCRBY 'anyLock' 'afc990dd-79b6-4d48-b07e-d900796d7691:48'  -1
local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1); 

# 大于0,代表还有其它的线程在使用(这里用到了重入锁)
if (counter > 0) then 
  # 重新设置Hash过期时间
  # PEXPIRE 'anyLock'  10000
  redis.call('pexpire', KEYS[1], ARGV[2]); 
  return 0; 
else 
  # 小于等于零的情况下,删除KEY,发布事件
  
  # DEL 'anyLock'
  redis.call('del', KEYS[1]); 
  
  # PUBLISH 'redisson_lock__channel:{anyLock}]' 0
  redis.call('publish', KEYS[2], ARGV[1]); 
  return 1; 
end; 

return nil;
```
### (5). 总结
> RedissonLock是通过Redis+Lua来实现分布式锁的,而且,还实现了重入锁.但是,RedissonLock是一个非公平锁,公平锁有排队功能,后面会继续进行解剖. 