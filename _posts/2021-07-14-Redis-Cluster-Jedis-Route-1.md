---
layout: post
title: 'Jedis客户端是如何把key与slot映射并路由(一)'
date: 2021-07-14
author: 李新
tags:  Redis 
---

### (1). 概述

Redis在集群模式下,会分成16384个slot,那么,我们使用Jedis时,是完全透明的,当我们初始化Jedis时,底层到底做了什么?  

### (2). 查看集群slot信息
> 先温习下这个命令.

```
127.0.0.1:6380> CLUSTER SLOTS
1) 1) (integer) 1366
   2) (integer) 5461
   3) 1) "127.0.0.1"
      2) (integer) 6380
      3) "d4fb72a249a42ecfce9f5e6d90eab917ddf20c22"
   4) 1) "127.0.0.1"
      2) (integer) 6385
      3) "edaeeffe57750afe67957f9ebbbcb0d919e9baea"
2) 1) (integer) 0
   2) (integer) 1365
   3) 1) "127.0.0.1"
      2) (integer) 6386
      3) "d5f9df749b601f5951c0f56ebfa810fe49b65543"
   4) 1) "127.0.0.1"
      2) (integer) 6387
      3) "6f1fdd73eef7c337e592ebbbd5eb7895ca8d8f56"
3) 1) (integer) 5462
   2) (integer) 8191
   3) 1) "127.0.0.1"
      2) (integer) 6386
      3) "d5f9df749b601f5951c0f56ebfa810fe49b65543"
   4) 1) "127.0.0.1"
      2) (integer) 6387
      3) "6f1fdd73eef7c337e592ebbbd5eb7895ca8d8f56"
4) 1) (integer) 8192
   2) (integer) 12287
   3) 1) "127.0.0.1"
      2) (integer) 6381
      3) "d36cf78e5481e488bc50dd31960044cc1e69522f"
   4) 1) "127.0.0.1"
      2) (integer) 6383
      3) "bf09356f37d20288d1a8cb5754b0f5b6afddfdae"
5) 1) (integer) 12288
   2) (integer) 16383
   3) 1) "127.0.0.1"
      2) (integer) 6382
      3) "8a9bf55091d15fd77de0bf7d4db05e09a5b682db"
   4) 1) "127.0.0.1"
      2) (integer) 6384
      3) "894425fde65f7da875d4d078f41e01381e52c6d9"
```
### (3). JedisConnectionFactory初始化
```
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
	
	// ***********************************************************
	// JedisConnectionFactory承接着连接池和配置信息.
	// ***********************************************************
	return new JedisConnectionFactory(redisClusterConfiguration, jedisPoolConfig);
}
```
### (4). JedisConnectionFactory
```
// JedisConnectionFactory实现了:InitializingBean,所以,在Spring容器初始化的时候,会调用:afterPropertiesSet方法

public void afterPropertiesSet() {
	if (shardInfo == null && clientConfiguration instanceof MutableJedisClientConfiguration) {
		// 
	}

	if (getUsePool() && !isRedisClusterAware()) {
		this.pool = createPool();
	}

	// 1. 判断是否集群模式
	if (isRedisClusterAware()) {
		this.cluster = createCluster();
	}
}// end afterPropertiesSet


private JedisCluster createCluster() {
	// 2. 创建JedisCluster
	JedisCluster cluster = createCluster((RedisClusterConfiguration) this.configuration, getPoolConfig());
	JedisClusterTopologyProvider topologyProvider = new JedisClusterTopologyProvider(cluster);
	this.clusterCommandExecutor = new ClusterCommandExecutor(topologyProvider,new JedisClusterConnection.JedisClusterNodeResourceProvider(cluster, topologyProvider), EXCEPTION_TRANSLATION);
	return cluster;
}

// 3. 具体创建:JedisCluster步骤
protected JedisCluster createCluster(RedisClusterConfiguration clusterConfig, GenericObjectPoolConfig poolConfig) {
	Assert.notNull(clusterConfig, "Cluster configuration must not be null!");
	Set<HostAndPort> hostAndPort = new HashSet<>();
	for (RedisNode node : clusterConfig.getClusterNodes()) {
		hostAndPort.add(new HostAndPort(node.getHost(), node.getPort()));
	}
	int redirects = clusterConfig.getMaxRedirects() != null ? clusterConfig.getMaxRedirects() : 5;
	int connectTimeout = getConnectTimeout();
	int readTimeout = getReadTimeout();
	
	// ****************************************************************
	// 有密码的情况下,配置密码,创建JedisCluster
	// 在这里,就会去获得所有的slot信息
	// ****************************************************************
	return StringUtils.hasText(getPassword())
			? new JedisCluster(hostAndPort, connectTimeout, readTimeout, redirects, getPassword(), poolConfig)
			: new JedisCluster(hostAndPort, connectTimeout, readTimeout, redirects, poolConfig);
}
```
### (5).  JedisCluster
```

// 1. JedisCluster的父类为:BinaryJedisCluster
public JedisCluster(Set<HostAndPort> jedisClusterNode, int connectionTimeout, int soTimeout,
                      int maxAttempts, String password, final GenericObjectPoolConfig poolConfig) {
	super(jedisClusterNode, connectionTimeout, soTimeout, maxAttempts, password, poolConfig);
}

// 2. BinaryJedisCluster内部会创建一个:JedisSlotBasedConnectionHandler
public BinaryJedisCluster(Set<HostAndPort> jedisClusterNode, int connectionTimeout, int soTimeout, int maxAttempts, String password, GenericObjectPoolConfig poolConfig) {
	this.connectionHandler = new JedisSlotBasedConnectionHandler(jedisClusterNode, poolConfig,connectionTimeout, soTimeout, password);
	this.maxAttempts = maxAttempts;
}
```
### (6). JedisSlotBasedConnectionHandler
```
// 1. JedisSlotBasedConnectionHandler的父类JedisClusterConnectionHandler
public JedisSlotBasedConnectionHandler(Set<HostAndPort> nodes, GenericObjectPoolConfig poolConfig, int connectionTimeout, int soTimeout, String password) {
	super(nodes, poolConfig, connectionTimeout, soTimeout, password);
}

// 2. JedisClusterConnectionHandler内部会持有:JedisClusterInfoCache
public JedisClusterConnectionHandler(Set<HostAndPort> nodes,final GenericObjectPoolConfig poolConfig, int connectionTimeout, int soTimeout, String password) {
	// 2.1 创建JedisClusterInfoCache
	this.cache = new JedisClusterInfoCache(poolConfig, connectionTimeout, soTimeout, password);
	// *************************************************************
	// 2.2 初始化slot
	// *************************************************************
	initializeSlotsCache(nodes, poolConfig, password);
}
```
### (7). JedisSlotBasedConnectionHandler.initializeSlotsCache
```
private void initializeSlotsCache(Set<HostAndPort> startNodes, GenericObjectPoolConfig poolConfig, String password) {
	// 1. 遍历所有的集群节点
	for (HostAndPort hostAndPort : startNodes) {
	  // 2. 创建Jedis
	  Jedis jedis = new Jedis(hostAndPort.getHost(), hostAndPort.getPort());
	  // 3. 配置密码
	  if (password != null) {
		jedis.auth(password);
	  }
	  
	  try {
		// *******************************************************************
		// 4. 调用JedisClusterInfoCache.discoverClusterNodesAndSlots方法发现集群节点和slot
		// *******************************************************************
		cache.discoverClusterNodesAndSlots(jedis);
		// 没有抛出异常的情况下,退出for循环,意思是:随机挑一个节点,能获取到slot信息即可.
		break;
	  } catch (JedisConnectionException e) {
		// try next nodes
		// 有异常的情况下,忽略
	  } finally {
		if (jedis != null) {
		  jedis.close();
		}
	  } //end finally
	}
}// end initializeSlotsCache
```
### (8). JedisClusterInfoCache
```
public class JedisClusterInfoCache {
  // key:ip:port  value:	JedisPool
  private final Map<String, JedisPool> nodes = new HashMap<String, JedisPool>();
  
  // ******************************************************************************
  // 注意:这里key会有16384个来着
  // key:0~16383  value:    JedisPool
  // ******************************************************************************
  private final Map<Integer, JedisPool> slots = new HashMap<Integer, JedisPool>();

  private final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
  private final Lock r = rwl.readLock();
  private final Lock w = rwl.writeLock();
  private volatile boolean rediscovering;
  private final GenericObjectPoolConfig poolConfig;

  private int connectionTimeout;
  private int soTimeout;
  private String password;

  private static final int MASTER_NODE_INDEX = 2;

  public void discoverClusterNodesAndSlots(Jedis jedis) {
	  w.lock();

	  try {
		// 1. 先清空一把nodes和slots
		reset();
		
		// 2. 执行:CLUSTER SLOTS
		List<Object> slots = jedis.clusterSlots();

		for (Object slotInfoObj : slots) {
		  List<Object> slotInfo = (List<Object>) slotInfoObj;
          
		  // 3. 当数量小于等于2时,不需要进行解析了.代表着这个slot里没有相应的master和slave
		  if (slotInfo.size() <= MASTER_NODE_INDEX) {
			continue;
		  }

		  // 4. 在这里会的内容是:start ~ end之间的slot,是一批
		  List<Integer> slotNums = getAssignedSlotArray(slotInfo);
			
		  // hostInfos
		  int size = slotInfo.size();
		  // 从第2个下标开始遍历数据(是master和slave数据)
		  for (int i = MASTER_NODE_INDEX; i < size; i++) {
			List<Object> hostInfos = (List<Object>) slotInfo.get(i);
			if (hostInfos.size() <= 0) {
			  continue;
			}
			
			// 5. 生成主机和端口信息
			HostAndPort targetNode = generateHostAndPort(hostInfos);
			
			// 6. 创建JedisPool对象
			// nodes.put("{ip:port}",JedisPool)
			setupNodeIfNotExist(targetNode);
			
			// 下标是第二个的时候,代表着是Master
			if (i == MASTER_NODE_INDEX) {
              // **********************************************************
			  // 7. 只有masster节点的情况下才会分配slot与master节点的映射关系.
			  // **********************************************************
			  assignSlotsToNode(slotNums, targetNode);
			}
		  }
		}
	  } finally {
		w.unlock();
	  }
	} // end discoverClusterNodesAndSlots
	
	
	public void assignSlotsToNode(List<Integer> targetSlots, HostAndPort targetNode) {
		w.lock();
		try {
		  // 看到没有?slot与JedisPool的关系
		  JedisPool targetPool = setupNodeIfNotExist(targetNode);
		  for (Integer slot : targetSlots) {
			// ***********************************************
			// 将slot与JedisPool进行映射.
			// ***********************************************
			slots.put(slot, targetPool);
		  }
		} finally {
		  w.unlock();
		}
	} // end assignSlotsToNode
}	
```

### (9). Jedis solt初始化
!["Jedis solt初始化"](/assets/redis/imgs/redis-solts.png)

### (10). 总结
Jedis客户端,会在初始化时,调用Redis集群中任意一台机器,执行:CLUSTER SLOTS,获得slot与Redis服务器之间的关系,并在内存中维护好slot与Redis实例之间的关系.  
在这一小节,剖析了,Jedis是如何初始化slot的,但是,有两个问题值得考虑:  
1) 调用Jedis进行保存操作时,底层是如何做的?    
2) slot迁移时,Jedis又是如何感知的?   