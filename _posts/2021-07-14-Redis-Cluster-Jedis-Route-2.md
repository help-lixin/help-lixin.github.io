---
layout: post
title: 'Jedis客户端是如何把key与slot映射并路由(二)'
date: 2021-07-14
author: 李新
tags:  Redis 
---

### (1). 概述
在前面,剖析了,当JedisConnectionFactory初始化时,底层会用Map<slot,JedisPool>来保存slot与JedisPool映射的关系,而,我们在使用Jedis执行set命令时,底层是如何处理的?  

### (2). Redis set操作
```
// 在深入源码之前,先要聊一件事:在Spring中我们有使用过JdbcTemplate,在这个类内部是要求配置:DataSource的.
// StringRedisTemplate在初始化时,实际也是需要一个:RedisConnectionFactory的.
// StringRedisTemplate在哪初始化的呢?  
// 答案就是:RedisAutoConfiguration.stringRedisTemplate

// 1. 获得StringRedisTemplate
StringRedisTemplate stringRedisTemplate = ctx.getBean(StringRedisTemplate.class);
// 2.  new BoundValueOperations
BoundValueOperations<String, String> ops = stringRedisTemplate.boundValueOps("c");
// 3. set操作
ops.set("cccc");
```
### (3). StringRedisTemplate
```
public BoundValueOperations<K, V> boundValueOps(K key) {
	// 每次调用,都new出来一个
	return new DefaultBoundValueOperations<>(key, this);
}// end BoundValueOperations

// ************************************************************
// StringRedisTemplate.boundValueOps 返回了一个:DefaultBoundValueOperations
// ************************************************************
DefaultBoundValueOperations(K key, RedisOperations<K, V> operations) {
	super(key, operations);
	// 回调:RedisTemplate.opsForValue
	this.ops = operations.opsForValue();
} // end DefaultBoundValueOperations

// RedisTemplate.opsForValue
public ValueOperations<K, V> opsForValue() {
	if (valueOps == null) {
		// *********************************************************
		// new DefaultValueOperations
		// *********************************************************
		valueOps = new DefaultValueOperations<>(this);
	}
	return valueOps;
} // end RedisTemplate.opsForValue
```
### (4). DefaultValueOperations
```
public void set(K key, V value) {
	// 1. 对value进行序列化
	byte[] rawValue = rawValue(value);
	
	// 3. 调用:execute
	execute(
	    // 2. 创建了一个Callback
	    new ValueDeserializingRedisCallback(key) {
			@Override
			protected byte[] inRedis(byte[] rawKey, RedisConnection connection) {
				connection.set(rawKey, rawValue);
				return null;
			}
	   }, 
	   true);
} // end set


// 4. 委托给:StringRedisTemplate去执行
<T> T execute(RedisCallback<T> callback, boolean b) {
	// **************************************************************
	// 调用StringRedisTemplate.execute方法,并传递一个Callback
	// **************************************************************
	return template.execute(callback, b);
}
```
### (5). StringRedisTemplate
```
// 1. execute
public <T> T execute(RedisCallback<T> action, boolean exposeConnection) {
	return execute(action, exposeConnection, false);
} 


public <T> T execute(
          //回调
          RedisCallback<T> action, 
		  // true
		  boolean exposeConnection, 
		  // false
		  boolean pipeline) {
	
	// 2. 获得RedisConnectionFactory
	RedisConnectionFactory factory = getRequiredConnectionFactory();
	RedisConnection conn = null;
	try {
		if (enableTransactionSupport) { //false
			// only bind resources in case of potential transaction synchronization
			conn = RedisConnectionUtils.bindConnection(factory, enableTransactionSupport);
		} else {
			// 创建:RedisConnection
			conn = RedisConnectionUtils.getConnection(factory);
		}
		
		// false
		boolean existingConnection = TransactionSynchronizationManager.hasResource(factory);
		// Spring留出来的,前置回调
		RedisConnection connToUse = preProcessConnection(conn, existingConnection);
		
		// 是否pipline
		// false
		boolean pipelineStatus = connToUse.isPipelined();
		if (pipeline && !pipelineStatus) {
			connToUse.openPipeline();
		}
		
		// exposeConnection : true,所以,直接用上面的连接:RedisConnection
		// 先不管,为什么要创建:Proxy了
		RedisConnection connToExpose = (exposeConnection ? connToUse : createRedisConnectionProxy(connToUse));
		
		// *************************************************************
		// 调用上面第4步(DefaultValueOperations.set)创建的:RedisCallback(ValueDeserializingRedisCallback为实现类),并传递一个:RedisConnection
		// *************************************************************
		T result = action.doInRedis(connToExpose);
		
		// 针对pipeline的处理.
		// close pipeline
		if (pipeline && !pipelineStatus) {
			connToUse.closePipeline();
		}
		
		// Spring留出来的后置处理.
		// TODO: any other connection processing?
		return postProcessResult(result, connToUse, existingConnection);
	} finally {
		// 释放conn
		RedisConnectionUtils.releaseConnection(conn, factory);
	}
} // end execute

```
### (6). ValueDeserializingRedisCallback
```
abstract class AbstractOperations<K, V> {
	abstract class ValueDeserializingRedisCallback implements RedisCallback<V> {
		private Object key;

		public ValueDeserializingRedisCallback(Object key) {
			this.key = key;
		}
		
		// 1. 调用:doInRedis
		public final V doInRedis(RedisConnection connection) {
			// ************************************************************
			// 2. rawKey方法,对key进行序列化.  
			// 3. 把RedisConnection和rawKey交给子类(DefaultValueOperations.set创建的回调)
			// ************************************************************
			byte[] result = inRedis(rawKey(key), connection);
			return deserializeValue(result);
		}

		@Nullable
		protected abstract byte[] inRedis(byte[] rawKey, RedisConnection connection);
	}
}
```
### (7). DefaultValueOperations
```
public void set(K key, V value) {
	byte[] rawValue = rawValue(value);
	execute(
	   
	   new ValueDeserializingRedisCallback(key) {
		// ***************************************************************
		// 1. ValueDeserializingRedisCallback回调inRedis
		// ***************************************************************
		@Override
		protected byte[] inRedis(byte[] rawKey, RedisConnection connection) {
			// 2. 最终是委托给了:RedisConnection
			// org.springframework.data.redis.connection.DefaultStringRedisConnection
			connection.set(rawKey, rawValue);
			return null;
		}
	}, true);
}
```
### (8). DefaultStringRedisConnection
```
public Boolean set(byte[] key, byte[] value) {
	// ******************************************************************
	// 1. 委托给:JedisClusterConnection.set方法
	//    JedisClusterConnection应该不陌生吧,在上一章里,有遇到过
	// ******************************************************************
	return convertAndReturn(delegate.set(key, value), identityConverter);
}
```
### (9). JedisClusterConnection
```
default Boolean set(byte[] key, byte[] value) {
	// ****************************************
	// 1. 创建:JedisClusterStringCommands
	// 2. 调用:JedisClusterStringCommands.set
	// ****************************************
	return stringCommands().set(key, value);
}

public RedisStringCommands stringCommands() {
	return new JedisClusterStringCommands(this);
}
```
### (10). JedisClusterStringCommands
```
public Boolean set(byte[] key, byte[] value) {
	Assert.notNull(value, "Value must not be null!");
	try {
		// 1. JedisClusterConnection.set
		return Converters.stringToBoolean(connection.getCluster().set(key, value));
	} catch (Exception ex) {
		throw convertJedisAccessException(ex);
	}
}
```
### (11). BinaryJedisCluster
```
public String set(final byte[] key, final byte[] value) {
	// 创建JedisClusterCommand,并调用:runBinary方法
	return new JedisClusterCommand<String>(connectionHandler, maxAttempts) {
	  @Override
	  public String execute(Jedis connection) {
		return connection.set(key, value);
	  }
	}.runBinary(key);
}
```
### (12). JedisClusterCommand
```
public T runBinary(byte[] key) {
	if (key == null) {
	throw new JedisClusterException("No way to dispatch this command to Redis Cluster.");
	}
	
	// **************************************************
	// runWithRetries
	// **************************************************
	return runWithRetries(key, this.maxAttempts, false, false);
}// end runBinary

private T runWithRetries(byte[] key, int attempts, boolean tryRandomNode, boolean asking) {
	if (attempts <= 0) {
	  throw new JedisClusterMaxRedirectionsException("Too many Cluster redirections?");
	}

	Jedis connection = null;
	try {

	  if (asking) {
		// TODO: Pipeline asking with the original command to make it
		// faster....
		connection = askConnection.get();
		connection.asking();

		// if asking success, reset asking flag
		asking = false;
	  } else {
		if (tryRandomNode) {
		  connection = connectionHandler.getConnection();
		} else {
			// ******************************************************
			// 1. 对key进行crc16算法
			// 2. 委托给:JedisSlotBasedConnectionHandler类的getConnectionFromSlot,根据slot获得对应的:Jedis
			// ******************************************************
		  connection = connectionHandler.getConnectionFromSlot(JedisClusterCRC16.getSlot(key));
		}
	  }

	  return execute(connection);

	} catch (JedisNoReachableClusterNodeException jnrcne) {
	  throw jnrcne;
	} catch (JedisConnectionException jce) {    // 连接异常时,进行重试,最大重试次数为:5
	  // release current connection before recursion
	  releaseConnection(connection);
	  connection = null;

	  if (attempts <= 1) {
		this.connectionHandler.renewSlotCache();
		throw jce;
	  }
	  return runWithRetries(key, attempts - 1, tryRandomNode, asking);
	} catch (JedisRedirectionException jre) {   // 重定向时处理.
	  // if MOVED redirection occurred,
	  if (jre instanceof JedisMovedDataException) {
		this.connectionHandler.renewSlotCache(connection);
	  }

	  // release current connection before recursion or renewing
	  releaseConnection(connection);
	  connection = null;

	  if (jre instanceof JedisAskDataException) {
		asking = true;
		askConnection.set(this.connectionHandler.getConnectionFromNode(jre.getTargetNode()));
	  } else if (jre instanceof JedisMovedDataException) {
	  } else {
		throw new JedisClusterException(jre);
	  }
	  return runWithRetries(key, attempts - 1, false, asking);
	} finally {
	  releaseConnection(connection);
	}
}
```
### (13). JedisClusterCRC16
```
// 在集群模式下,如果key包含有花括号,那么会根据花括号的内容计算,而不是整个key.
public static int getSlot(byte[] key) {
	int s = -1;
	int e = -1;
	boolean sFound = false;
	for (int i = 0; i < key.length; i++) {
	  if (key[i] == '{' && !sFound) {
		s = i;
		sFound = true;
	  }
	  if (key[i] == '}' && sFound) {
		e = i;
		break;
	  }
	}
	if (s > -1 && e > -1 && e != s + 1) {
	  return getCRC16(key, s + 1, e) & (16384 - 1);
	}
	return getCRC16(key) & (16384 - 1);
}
```
### (14). JedisSlotBasedConnectionHandler
```
public Jedis getConnectionFromSlot(int slot) {
	// 委托给:JedisClusterInfoCache
    JedisPool connectionPool = cache.getSlotPool(slot);
    if (connectionPool != null) {
      return connectionPool.getResource();
    } else {
      renewSlotCache(); //It's abnormal situation for cluster mode, that we have just nothing for slot, try to rediscover state
      connectionPool = cache.getSlotPool(slot);
      if (connectionPool != null) {
        return connectionPool.getResource();
      } else {
        //no choice, fallback to new connection to random node
        return getConnection();
      }
    }
}
```
### (15). JedisClusterInfoCache
```
// ***************************************************************
// 这个类就是上一节,剖析的重点,在启动时,会把所有的slot进行初始化.
// map的key就是:slot
// map的value就是:JedisPool
// ***************************************************************
private final Map<Integer, JedisPool> slots = new HashMap<Integer, JedisPool>();
public JedisPool getSlotPool(int slot) {
    r.lock();
    try {
      return slots.get(slot);
    } finally {
      r.unlock();
    }
}
```
### (16). 总结
1) Jedis启动时,会初始化slot与JedisPool(通过Map保存着).         
2) Jedis在保存数据时(set),会先对key进行CRC16并与16384进行运算,得出slot,通过slot获得:JedisPool.        
3) Jedis在根据slot获得:Jedis实例.              
4) 在获取实例时,如果有抛出:JedisConnectionException或JedisRedirectionException时,会触发:JedisClusterInfoCache去与Redis同步(CLUSTER SLOTS)slot信息.   