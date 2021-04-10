---
layout: post
title: 'Calcite Statement执行SQL(三)'
date: 2021-04-11
author: 李新
tags:  Calcite源码
---

### (1). 概述
> 跳过跟踪:Statement创建的过程,我们直接看:Statement.executeQuery("select * from EMPS"),是如何做的.  
> Calcite对Statement的实现类是:CalciteJdbc41Statement(是通过:CalciteJdbc41Factory.newStatement构建出来的).  

### (2). CalciteJdbc41Statement.executeQuery
```
public ResultSet executeQuery(String sql) throws SQLException {
	// 检查是否关闭
    checkOpen();
	// 检查是否为:PreparedStatement或CallableStatement,如果是这两个就抛出异常.
	// 因为:PreparedStatement和CallableStatement创建方法是不一样的.
    checkNotPreparedOrCallable("executeQuery(String)");
    try {
	
	 // 1. 委托给:executeInternal方法进行调用
      executeInternal(sql);
	  
      if (openResultSet == null) {
        throw AvaticaConnection.HELPER.createException(
            "Statement did not return a result set");
      } 
      return openResultSet;
    } catch (RuntimeException e) {
      // ... ... 
    } 
}// end executeQuery

protected void executeInternal(String sql) throws SQLException {
    // 
    this.updateCount = -1;
    try {
	  // limit配置,为:"0"时,不做分页
      // In JDBC, maxRowCount = 0 means no limit; in prepare it means LIMIT 0
      final long maxRowCount1 = maxRowCount <= 0 ? -1 : maxRowCount;
	  // 当查询失败时,尝试N次进行查询(默认:N=5)
      for (int i = 0; i < connection.maxRetriesPerExecute; i++) {
        try {
		   // *********************************************************
		   // 2. 委托给了:java.sql.Connection的实现类(org.apache.calcite.avatica.AvaticaJdbc41Connection)去执行:SQL
		  // *********************************************************
          @SuppressWarnings("unused")
          Meta.ExecuteResult x =
              connection.prepareAndExecuteInternal(this, sql, maxRowCount1);
          return;
        } catch (NoSuchStatementException e) {
          resetStatement();
        }
      }
    } catch (RuntimeException e) {
      // ... ... 
    }
    // ... ... 
} // end executeInternal
```
### (3). AvaticaJdbc41Connection.prepareAndExecuteInternal
```
protected Meta.ExecuteResult prepareAndExecuteInternal(
      final AvaticaStatement statement, final String sql, long maxRowCount)
      throws SQLException, NoSuchStatementException {
	
	// 1. 创建:Meta.PrepareCallback
    final Meta.PrepareCallback callback =
        new Meta.PrepareCallback() {
          public Object getMonitor() {
            return statement;
          }

          public void clear() throws SQLException {
            if (statement.openResultSet != null) {
              final AvaticaResultSet rs = statement.openResultSet;
              statement.openResultSet = null;
              try {
                rs.close();
              } catch (Exception e) {
                throw HELPER.createException(
                    "Error while closing previous result set", e);
              }
            }
          }

          public void assign(Meta.Signature signature, Meta.Frame firstFrame,
              long updateCount) throws SQLException {
            statement.setSignature(signature);

            if (updateCount != -1) {
              statement.updateCount = updateCount;
            } else {
              final TimeZone timeZone = getTimeZone();
              statement.openResultSet = factory.newResultSet(statement, new QueryState(sql),
                  signature, timeZone, firstFrame);
            }
          }

          public void execute() throws SQLException {
            if (statement.openResultSet != null) {
              statement.openResultSet.execute();
              isUpdateCapable(statement);
            }
          }
        };
    
	// *********************************************************
	// 2. 通过Meta(CalciteMetaImpl),去执行:sql(并设置了:Meta.PrepareCallback函数)
	// *********************************************************
    return meta.prepareAndExecute(statement.handle, sql, maxRowCount,
        AvaticaUtils.toSaturatedInt(maxRowCount), callback);
}
```
### (4). CalciteMetaImpl.prepareAndExecute
```
public ExecuteResult prepareAndExecute(
                          StatementHandle h,
						  String sql, 
						  long maxRowCount, 
						  int maxRowsInFirstFrame,
						  // 回调函数
						  PrepareCallback callback) throws NoSuchStatementException {
							  
    final CalcitePrepare.CalciteSignature<Object> signature;
    try {
      final int updateCount;
	  //  1. 先调用回调函数的:getMonitor方法,这个方法返回的是:StatementHandle(CalciteJdbc41Statement),给这个对象上锁
      synchronized (callback.getMonitor()) { 
		  // 2. 再调用回调函数:clear方法,应该是先清场吧!
        callback.clear();
		
		// 3. 获得:java.sql.Connection的实现类(CalciteJdbc41Connection)
        final CalciteConnectionImpl calciteConnection = getConnection();
		
		// 4. 获得:CalciteServerStatement
        final CalciteServerStatement statement = calciteConnection.server.getStatement(h);
		
		// 5. 通过CalciteJdbc41Connection创建一个: CalciteJdbc41Connection$ContextImpl
        final Context context = statement.createPrepareContext();
		
		// 6. 开始进行query
        final CalcitePrepare.Query<Object> query = toQuery(context, sql);
        signature = calciteConnection.parseQuery(query, context, maxRowCount);
		
		
        statement.setSignature(signature);
        switch (signature.statementType) {
        case CREATE:
        case DROP:
        case ALTER:
        case OTHER_DDL:
          updateCount = 0; // DDL produces no result set
          break;
        default:
          updateCount = -1; // SELECT and DML produces result set
          break;
        }
        callback.assign(signature, null, updateCount);
      }
      callback.execute();
      final MetaResultSet metaResultSet =
          MetaResultSet.create(h.connectionId, h.id, false, signature, null, updateCount);
      return new ExecuteResult(ImmutableList.of(metaResultSet));
    } catch (SQLException e) {
      throw new RuntimeException(e);
    }
} // end prepareAndExecute
```
### (5). 
