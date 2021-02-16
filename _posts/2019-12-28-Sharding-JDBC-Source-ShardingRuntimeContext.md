---
layout: post
title: 'Sharding-JDBC ShardingRuntimeContext(六)'
date: 2019-12-28
author: 李新
tags: Sharding-JDBC源码
---

### (1). 概述
> 在这里,主要对:ShardingRuntimeContext类的职责进行了解.

### (2). 看下ShardingRuntimeContext的类图
!["ShardingRuntimeContext类图"](/assets/sharding-jdbc/imgs/ShardingRuntimeContext.jpg)

### (3). ShardingRuntimeContext构建时机?
> ShardingRuntimeContext是在:ShardingDataSource初始化时,构建的.

```
public ShardingDataSource(final Map<String, DataSource> dataSourceMap, final ShardingRule shardingRule, final Properties props) throws SQLException {
	//... ...
	runtimeContext = new ShardingRuntimeContext(dataSourceMap, shardingRule, props, getDatabaseType());
}
```
### (4). ShardingRuntimeContext
> 初始化子类之前,会先初始化父类.  

```
// 1. AbstractRuntimeContext构造器初始化
// domain
// T == ShardingRule
private final T rule;
private final ConfigurationProperties properties;
private final DatabaseType databaseType;
// 执行引擎
private final ExecutorEngine executorEngine;
// SQL解析引擎
private final SQLParserEngine sqlParserEngine;

 protected AbstractRuntimeContext(final T rule, final Properties props, final DatabaseType databaseType) {
	 // ShardingRule
	this.rule = rule;
	properties = new ConfigurationProperties(null == props ? new Properties() : props);
	this.databaseType = databaseType;
	// 执行引擎(串行/平行/同步/异步执行,内部实际为一个线程池)
	executorEngine = new ExecutorEngine(properties.<Integer>getValue(ConfigurationPropertyKey.EXECUTOR_SIZE));
	// 根据:databaseTypeName创建:SQLParserEngine,并缓存.
	sqlParserEngine = SQLParserEngineFactory.getSQLParserEngine(DatabaseTypes.getTrunkDatabaseTypeName(databaseType));
	// 打印日志信息
	ConfigurationLogger.log(rule.getRuleConfiguration());
	ConfigurationLogger.log(props);
}// end 

// 2. MultipleDataSourcesRuntimeContext初始化
private final ShardingSphereMetaData metaData;

protected MultipleDataSourcesRuntimeContext(final Map<String, DataSource> dataSourceMap, final T rule, final Properties props, final DatabaseType databaseType) throws SQLException {
	// 调用父类:AbstractRuntimeContext的构造器
	super(rule, props, databaseType);
	// 对java.sql.DatabaseMetaData进行封装,转换成:ShardingSphereMetaData
	metaData = createMetaData(dataSourceMap, databaseType);
} // end

// 3. ShardingRuntimeContext初始化
// 缓存:java.sql.DatabaseMetaData
private final CachedDatabaseMetaData cachedDatabaseMetaData;
// 事务管理器
private final ShardingTransactionManagerEngine shardingTransactionManagerEngine;

public ShardingRuntimeContext(final Map<String, DataSource> dataSourceMap, final ShardingRule shardingRule, final Properties props, final DatabaseType databaseType) throws SQLException {
	// 先初始化父类:MultipleDataSourcesRuntimeContext的构造器
	super(dataSourceMap, shardingRule, props, databaseType);
	// 对java.sql.DatabaseMetaData进行封装,转换成:ShardingSphereMetaData进行缓存.
	cachedDatabaseMetaData = createCachedDatabaseMetaData(dataSourceMap);
	
	// ************************************************************************
	// 5. 创建事务管理引擎,ShardingTransactionManagerEngine构造器
	//    会通过SPI,加载:ShardingTransactionManager类的所有实现类.
	// ************************************************************************
	shardingTransactionManagerEngine = new ShardingTransactionManagerEngine();
	shardingTransactionManagerEngine.init(databaseType, dataSourceMap);
} // end
```
### (5). ShardingTransactionManagerEngine
```
public final class ShardingTransactionManagerEngine {
    // Cache
    private final Map<TransactionType, ShardingTransactionManager> transactionManagerMap = new EnumMap<>(TransactionType.class);
    
    public ShardingTransactionManagerEngine() {
		// 加载事务管理器
        loadShardingTransactionManager();
    }// end ShardingTransactionManagerEngine
    
    private void loadShardingTransactionManager() {
		// 通过SPI加载:ShardingTransactionManager的所有实现.
        for (ShardingTransactionManager each : ServiceLoader.load(ShardingTransactionManager.class)) {
            if (transactionManagerMap.containsKey(each.getTransactionType())) {
                log.warn("Find more than one {} transaction manager implementation class, use `{}` now",
                    each.getTransactionType(), transactionManagerMap.get(each.getTransactionType()).getClass().getName());
                continue;
            }
			// 设置到缓存中
            transactionManagerMap.put(each.getTransactionType(), each);
        }
    } // end loadShardingTransactionManager
	
	// ... ...
}
```
### (6). 总结
> 从现在的分析结果来看,ShardingRuntimeContext的职责如下:  
> 1. 对SQL进行解析(SQLParserEngine).  
> 2. 执行任务(线程池/ExecutorEngine).     
> 3. 事务管理(ShardingTransactionManager).  