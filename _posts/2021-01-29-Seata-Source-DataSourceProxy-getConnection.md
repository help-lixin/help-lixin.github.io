---
layout: post
title: 'Seata 分支事务处理之DataSourceProxy(三)'
date: 2021-01-29
author: 李新
tags: Seata源码
---

### (1). AT模式下切入口在呢?
> AT模式的切入口在:  
> 1. 以JDBC编程为例(Connection/PreparedStatement/ResultSet).   
> 2. 业务端(JdbcTemplate/MyBatis/Hibernate...),会调用DataSourceProxy.getConnection()方法获取一个Connection,不过这个Connection是经过代理之后的(ConnectionProxy).  
> 3. 通过ConnectionProxy创建:PreparedStatementProxy(Statement).  
> 4. PreparedStatementProxy.execute返回:ResultSet.   

### (2). 测试发送请求
> 1. curl http://localhost:8084/purchase/commit
> 2. 断点在:AbstractConnectionProxy.prepareStatement

### (3). ConnectionProxy.getConnection
> 首先业务端(JdbcTemplate/MyBatis/Hibernate...)会获得一个Connection.  

```
public ConnectionProxy getConnection() throws SQLException {
	Connection targetConnection = targetDataSource.getConnection();
	return new ConnectionProxy(this, targetConnection);
}
```
### (4). AbstractConnectionProxy.prepareStatement
> 再次,业务端(JdbcTemplate/MyBatis/Hibernate...)会调用:Connection.prepareStatement方法.

```
public PreparedStatement prepareStatement(String sql) throws SQLException {
	// mysql
	String dbType = getDbType();
	// support oracle 10.2+
	
	PreparedStatement targetPreparedStatement = null;
	
	// 如果是AT模式下
	if (BranchType.AT == RootContext.getBranchType()) { // true
		// sql = update storage_tbl set count = count - ? where commodity_code = ?
		// 对SQL语句进行识别(io.seata.sqlparser.druid.mysql.MySQLUpdateRecognizer)
		List<SQLRecognizer> sqlRecognizers = SQLVisitorFactory.get(sql, dbType);
		
		if (sqlRecognizers != null && sqlRecognizers.size() == 1) {
			SQLRecognizer sqlRecognizer = sqlRecognizers.get(0);
			
			// 只有INSERT语句才会进入该逻辑
			if (sqlRecognizer != null && sqlRecognizer.getSQLType() == SQLType.INSERT) {
				TableMeta tableMeta = TableMetaCacheFactory.getTableMetaCache(dbType).getTableMeta(getTargetConnection(),
						sqlRecognizer.getTableName(), getDataSourceProxy().getResourceId());
				String[] pkNameArray = new String[tableMeta.getPrimaryKeyOnlyName().size()];
				tableMeta.getPrimaryKeyOnlyName().toArray(pkNameArray);
				targetPreparedStatement = getTargetConnection().prepareStatement(sql,pkNameArray);
			}// end if
		}// end if
	}// end if
	
	// update/delete语句都会进该逻辑
	if (targetPreparedStatement == null) { // true
		// 调用真实的:Connection.prepareStatement(sql)
		// getTargetConnection() = com.mysql.cj.jdbc.ConnectionImpl
		targetPreparedStatement = getTargetConnection().prepareStatement(sql);
	}
	
	// ****************************************************
	// 创建了:PreparedStatementProxy
	// ****************************************************
	return new PreparedStatementProxy(this, targetPreparedStatement, sql);
} //end prepareStatement
```
### (5). PreparedStatementProxy.executeUpdate
```
public int executeUpdate() throws SQLException {
	// 委派给了:ExecuteTemplate
	return ExecuteTemplate.execute(this, (statement, args) -> statement.executeUpdate());
}
```
### (6). ExecuteTemplate.execute
```
public class ExecuteTemplate {
	
	// 1. execute
	public static <T, S extends Statement> T execute(
			StatementProxy<S> statementProxy,
			StatementCallback<T, S> statementCallback,
			Object... args) throws SQLException {
		 //2. execute
		return execute(null, statementProxy, statementCallback, args);
	}// end execute
	
	
	// 3. execute
	public static <T, S extends Statement> T execute(
			// sqlRecognizers = null
			List<SQLRecognizer> sqlRecognizers,
			// statementProxy = io.seata.rm.datasource.PreparedStatementProxy
			StatementProxy<S> statementProxy,
			// 在第5步的lambad函数
			// io.seata.rm.datasource.PreparedStatementProxy$$Lambda$586/
			StatementCallback<T, S> statementCallback,
			// args=[]
			Object... args) throws SQLException {
			
			// 执行原始的SQL语句
			// false
	        if (!RootContext.requireGlobalLock() && BranchType.AT != RootContext.getBranchType()) {
	            // Just work as original statement
	            return statementCallback.execute(statementProxy.getTargetStatement(), args);
	        }
	
			// mysql
	        String dbType = statementProxy.getConnectionProxy().getDbType();
	        if (CollectionUtils.isEmpty(sqlRecognizers)) { //true
				// 对SQL语句又重新进行了一次识别?
				// 前面不是识别了吗?
				// 因为没办法通过参数传递过来
	            sqlRecognizers = SQLVisitorFactory.get(
	                    statementProxy.getTargetSQL(),
	                    dbType);
	        }
			
			// 执行器
	        Executor<T> executor;
			
	        if (CollectionUtils.isEmpty(sqlRecognizers)) { 
				// **************************************************
				// 如果对SQL语句没有识别到类型,那么,就使用默认的:PlainExecutor
				// **************************************************
	            executor = new PlainExecutor<>(statementProxy, statementCallback);
	        } else {
	            if (sqlRecognizers.size() == 1) {
	                SQLRecognizer sqlRecognizer = sqlRecognizers.get(0);
					
	                switch (sqlRecognizer.getSQLType()) {
	                    case INSERT:    // INSERT:
	                        executor = EnhancedServiceLoader.load(InsertExecutor.class, dbType,
	                                new Class[]{StatementProxy.class, StatementCallback.class, SQLRecognizer.class},
	                                new Object[]{statementProxy, statementCallback, sqlRecognizer});
	                        break;
	                    case UPDATE:   // UpdateExecutor
	                        executor = new UpdateExecutor<>(statementProxy, statementCallback, sqlRecognizer);
	                        break;
	                    case DELETE:    // DeleteExecutor
	                        executor = new DeleteExecutor<>(statementProxy, statementCallback, sqlRecognizer);
	                        break;
	                    case SELECT_FOR_UPDATE:
	                        executor = new SelectForUpdateExecutor<>(statementProxy, statementCallback, sqlRecognizer);
	                        break;
	                    default:
							// 默认:PlainExecutor
	                        executor = new PlainExecutor<>(statementProxy, statementCallback);
	                        break;
	                }
	            } else {
					// 如果识别出来的是多条SQL语句类型,那么执行器则是:MultiExecutor
	                executor = new MultiExecutor<>(statementProxy, statementCallback, sqlRecognizers);
	            }
	        } // end else
			
			
	        T rs;
	        try {
				// ************************************************************
				// 调用Executor去执行,并返回:ResultSet
				// 在这一篇我只能分析到这里.
				// Executor如何执行的,我会另开几篇进行分析.
				// ************************************************************
	            rs = executor.execute(args);
	        } catch (Throwable ex) {
	            if (!(ex instanceof SQLException)) {
	                // Turn other exception into SQLException
	                ex = new SQLException(ex);
	            }
	            throw (SQLException) ex;
	        }
	        return rs;
	    }// end execute
	
}
```
### (7). Seata AT模型执行流程图解
!["Seata AT模型执行流程图解"](/assets/seata/imgs/seata-Executor-Sequence.jpg)
### (8). 总结
> AT模型是以DataSource为切入点,对Connection/PreparedStatement进行了代理.   