---
layout: post
title: 'Sharding-JDBC ShardingConnection(七)'
date: 2019-12-28
author: 李新
tags: Sharding-JDBC源码
---

### (1). 概述
> 最终于了ShardingConnection的剖析了.   

### (2). ShardingConnection的构建时机?
> ShardingConnection是由应用程序构建的,只是我们平时用了一些ORM后,会无感知而已.  
> 切入点代码在:DataSource(ShardingDataSource).getConnection()方法.

```
public final ShardingConnection getConnection() {
	return new ShardingConnection(
		//  所有的数据源信息
	    getDataSourceMap(), 
		// ShardingRuntimeContext上下文
		runtimeContext, 
		// 在前面,稍微提到过:ShardingTransactionTypeScanner会根据注解:@ShardingTransactionType
		// 创建拦截器:ShardingTransactionTypeInterceptor,对方法进行拦截,所以:
		// 当然,如果没有走拦截器,默认是:TransactionType.LOCAL
		TransactionTypeHolder.get());
}
```
### (3). Connection(ShardingConnection)构造器
> ShardingConnection实现了Connection,所以,ShardingConnection具有Connection的特性.  
> 比如: createStatement/prepareStatement/prepareCall...   
> 所以,我们下一步是要跟踪:ShardingConnection.prepareStatement()方法.   

```
public ShardingConnection(final Map<String, DataSource> dataSourceMap, final ShardingRuntimeContext runtimeContext, final TransactionType transactionType) {
	this.dataSourceMap = dataSourceMap;
	this.runtimeContext = runtimeContext;
	// TransactionType.LOCAL
	this.transactionType = transactionType;
	// 获得事管理器
	shardingTransactionManager = runtimeContext.getShardingTransactionManagerEngine().getTransactionManager(transactionType);
} // end ShardingConnection
```
### (4). Connection(ShardingConnection).prepareStatement
```
// 这里的内容,汲及到:PreparedStatement,另开一篇来讲,我这里主要以SELECT语句来分析.
// select *  from t_order t  where t.order_id in   (   ?   ,  ?   ,  ?   ,  ?   )
public PreparedStatement prepareStatement(final String sql) throws SQLException {
	return new ShardingPreparedStatement(this, sql);
}
```
### (5). 总结
> ShardingConnection由于实现了:Connection,所以,它具有Connection的职责.  
> ShardingConnection.prepareStatement(sql)该方法,另开一小节来剖析.   