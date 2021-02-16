---
layout: post
title: 'Sharding-JDBC ShardingDataSource(五)'
date: 2019-12-28
author: 李新
tags: Sharding-JDBC源码
---

### (1). 概述
> 在这一小节,主要剖析:ShardingDataSource,它是DataSource的实现类,也是应用程序与数据源的重要桥梁.  

### (2). ShardingDataSource何时创建的?
```
public static DataSource createDataSource(
		final Map<String, DataSource> dataSourceMap, 
		final ShardingRuleConfiguration shardingRuleConfig, 
		final Properties props) throws SQLException {
	// 2. 创建:ShardingDataSource对象,持有:ShardingRule/dataSourceMap
	return new ShardingDataSource(
		dataSourceMap, 
		// 1. 对ShardingRuleConfiguration数据解析,转换成:ShardingRule
		new ShardingRule(shardingRuleConfig, dataSourceMap.keySet()), 
		props);
}
```
### (3). 看下ShardingDataSource类图
!["ShardingDataSource类图"](/assets/sharding-jdbc/imgs/ShardingDataSource-Class-Diagram.jpg)

### (4). ShardingDataSource
```
public class ShardingDataSource extends AbstractDataSourceAdapter {
		
	// 上下文对象,内部持有: ShardingRule对象.
	// 留个疑问:
	// ShardingDataSource对象在整个jvm只有一份实例,这里是否会存在问题.后述再详细分析:ShardingRuntimeContext对象.
    private final ShardingRuntimeContext runtimeContext;
    
	// 1. 加载类之前,先加载静态方法.
    static {
		// 通过SPI加载:RouteDecorator/SQLRewriteContextDecorator/ResultProcessEngine
        NewInstanceServiceLoader.register(RouteDecorator.class);
        NewInstanceServiceLoader.register(SQLRewriteContextDecorator.class);
        NewInstanceServiceLoader.register(ResultProcessEngine.class);
    }// end static function
	
	
	public ShardingDataSource(final Map<String, DataSource> dataSourceMap, final ShardingRule shardingRule, final Properties props) throws SQLException {
		// 2. 把数据源集合交给父类
		super(dataSourceMap);
		// 检查数据源的类型(获取Connection上的URL,用以判断:数据源属于什么类型)
		// 我不太理解的是:dataSourceMap是一个集合,遍历检查可以,为什么最后还要Hold住遍历的最后一个:DatabaseType.
		// 假如集合,包含有:MySQL/Oracle,最后Hold住的是:DatabaseType靠谱吗?  
		checkDataSourceType(dataSourceMap);
		// 创建:ShardingRuntimeContext Hold住ShardingRule对象.
		runtimeContext = new ShardingRuntimeContext(dataSourceMap, shardingRule, props, getDatabaseType());
	} // end 
}
```
### (5). 总结 
> 在这里,我只对ShardingDataSource的构建过程和它的职责进行了分析,至于:调用过程另外再剖析.  
> 从ShardingDataSource的类图,我们就能看出:Sharding-JDBC对DataSource进行了封装.  