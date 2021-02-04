---
layout: post
title: 'Seata 分支事务处理之UpdateExecutor(四)'
date: 2021-01-29
author: 李新
tags: Seata-AT源码
---

### (1). 看下Executor类的实现
!["Seata Executor"](/assets/seata/imgs/seata-Executor.jpg)

### (2). 看下UpdateExecutor类的继承图
> 从前面分析到,我的SQL语句(为Update),所以,在这里只跟踪:UpdateExecutor

!["Seata UpdateExecutor类图"](/assets/seata/imgs/seata-Executor-Class-Diagram.jpg)

### (3). UpdateExecutor(BaseTransactionalExecutor).execute运行过程

```
public T execute(Object... args) throws Throwable {
	// 需要申明一下:
	// ****************************************************************
	// DataSourceProxy.getConnection()时,每次都是new ConnectionProxy
	// 所以,对ConnectionProxy的任何操作,都不会存在线程安全问题或环境污染问题.
	// 比如:statementProxy.getConnectionProxy().bind(xxx);
	// 比如: statementProxy.getConnectionProxy().setGlobalLockRequire(xxx)
	// ****************************************************************
	
	// 从ThreadLocal(RootContext)中获得xid
	// 我这里是分支事务,xid通过restTemplate传过来,又通过:mvc interceptor绑定到了ThreadLocal里了
	// 所以xid是有值的
	String xid = RootContext.getXID();
	if (xid != null) {
		// 设置xid
		statementProxy.getConnectionProxy().bind(xid);
	}
	
	// 设置是否全局锁
	// false
	statementProxy.getConnectionProxy().setGlobalLockRequire(RootContext.requireGlobalLock());
	// ******************************************************************
	// 4. 委托给子类:AbstractDMLBaseExecutor.doExecute
	// ******************************************************************
	return doExecute(args);
}
```
### (4). AbstractDMLBaseExecutor.doExecute
```
public T doExecute(Object... args) throws Throwable {
	// 4.1. 获得ConnectionProxy
	AbstractConnectionProxy connectionProxy = statementProxy.getConnectionProxy();
	// 4.2 判断Connection是否自动commit
	// 这个可以在连接池配置的
	if (connectionProxy.getAutoCommit()) { // 我这里是jdbc,所以:autocommit=true
		// 4.4 executeAutoCommitTrue对executeAutoCommitFalse方法进行了包装
		// 所以,专心研究:executeAutoCommitTrue即可.
		return executeAutoCommitTrue(args);
	} else {  
		return executeAutoCommitFalse(args);
	}
}// end doExecute


protected T executeAutoCommitTrue(Object[] args) throws Throwable {
	ConnectionProxy connectionProxy = statementProxy.getConnectionProxy();
	try {
		// 4.5 设置Connection autoCommit=false
		connectionProxy.setAutoCommit(false);
		
		// 4.6 创建:LockRetryPolicy对象,包裹着:ConnectionProxy对象.
		// 4.7 执行:LockRetryPolicy.execute()方法.
		return new LockRetryPolicy(connectionProxy).execute(() -> {
			// ***********************************************************
			// 4.8 执行:executeAutoCommitFalse方法
			// 5. 这部份的内容留到后面讲.
			// ***********************************************************
			T result = executeAutoCommitFalse(args);
			
			// 4.9 手动调用commit
			// *************************************************************
			// 这里很重要:
			// 1. 注册分支事务(获取记录行锁,seata称之为:全局锁)
			// 2. undolog刷盘
			// 3. 真正的commit.
			// connectionProxy.commit()我会专门留一节去分析,在这一节,不做分析.
			// *************************************************************
			connectionProxy.commit();
			// 返回结果
			return result;
		});
	} catch (Exception e) {
		// when exception occur in finally,this exception will lost, so just print it here
		LOGGER.error("execute executeAutoCommitTrue error:{}", e.getMessage(), e);
		if (!LockRetryPolicy.isLockRetryPolicyBranchRollbackOnConflict()) {
			// *****************************************************************
			// 此处很重要:
			// 在try失败的情况下,会向:TC汇报失败.
			// *****************************************************************
			connectionProxy.getTargetConnection().rollback();
		}
		throw e;
	} finally {
		
		// 在上面的new LockRetryPolicy(xxxx).execute()方法执行之后.
		// 1. 重置上下文(xid,id...)全部清除
		connectionProxy.getContext().reset();
		
		// 2. 重新设置autoCommit为true
		connectionProxy.setAutoCommit(true);
	}
}// end executeAutoCommitTrue
```
### (5). AbstractDMLBaseExecutor.executeAutoCommitFalse
```
protected T executeAutoCommitFalse(Object[] args) throws Exception {
	// 不是MySQL数据库,并且,有多个PK主键,抛出异常
	if (!JdbcConstants.MYSQL.equalsIgnoreCase(getDbType()) && isMultiPk()) {
		throw new NotSupportYetException("multi pk only support mysql!");
	}
	
	// ********************************************************************
	// 6.产生:UpdateExecutor.beforeImage
	// ********************************************************************
	TableRecords beforeImage = beforeImage();
	
	// ********************************************************************
	// 执行业务SQL
	// 通过真实的:PreparedStatemtn去执行业务SQL
	// statementProxy.getTargetStatement() = com.mysql.cj.jdbc.ClientPreparedStatement
	// ********************************************************************
	T result = statementCallback.execute(statementProxy.getTargetStatement(), args);
	
	// ********************************************************************
	// 7. 产生:UpdateExecutor.afterImage
	// ********************************************************************
	TableRecords afterImage = afterImage(beforeImage);
	
	// 8. 产生:undoLog
	//    BaseTransactionalExecutor.prepareUndoLog
	prepareUndoLog(beforeImage, afterImage);
	
	return result;
}
```
### (6). UpdateExecutor.beforeImage
```
protected TableRecords beforeImage() throws SQLException {
	ArrayList<List<Object>> paramAppenderList = new ArrayList<>();
	// 6.1. 获得表的元数据信息
	TableMeta tmeta = getTableMeta();
	// 6.2. 构建beforeImageSQL
	// 这是我的业务服务(store)生成的镜像前的SQL语句: 
	// SELECT id, count FROM storage_tbl WHERE commodity_code = ? FOR UPDATE
	String selectSQL = buildBeforeImageSQL(tmeta, paramAppenderList);
	
	//  把SQL转换成成业务模型:TableRecords
	return buildTableRecords(tmeta, selectSQL, paramAppenderList);
}// end beforeImage

// 6.2 构建镜像
private String buildBeforeImageSQL(TableMeta tableMeta, ArrayList<List<Object>> paramAppenderList) {
	
	SQLUpdateRecognizer recognizer = (SQLUpdateRecognizer) sqlRecognizer;
	List<String> updateColumns = recognizer.getUpdateColumns();
	assertContainsPKColumnName(updateColumns);
	
	StringBuilder prefix = new StringBuilder("SELECT ");
	StringBuilder suffix = new StringBuilder(" FROM ").append(getFromTableInSQL());
	String whereCondition = buildWhereCondition(recognizer, paramAppenderList);
	if (StringUtils.isNotBlank(whereCondition)) {
		suffix.append(WHERE).append(whereCondition);
	}
	String orderBy = recognizer.getOrderBy();
	if (StringUtils.isNotBlank(orderBy)) {
		suffix.append(orderBy);
	}
	ParametersHolder parametersHolder = statementProxy instanceof ParametersHolder ? (ParametersHolder)statementProxy : null;
	String limit = recognizer.getLimit(parametersHolder, paramAppenderList);
	if (StringUtils.isNotBlank(limit)) {
		suffix.append(limit);
	}
	// *******************************************************
	// 6.3 Seata为了解决脏读问题,用到了:FOR UPDATE.
	// 此时针对这行数据的并发写,变成了排队串行化了.
	// *******************************************************
	suffix.append(" FOR UPDATE");
	
	
	StringJoiner selectSQLJoin = new StringJoiner(", ", prefix.toString(), suffix.toString());
	if (ONLY_CARE_UPDATE_COLUMNS) {
		if (!containsPK(updateColumns)) {
			selectSQLJoin.add(getColumnNamesInSQL(tableMeta.getEscapePkNameList(getDbType())));
		}
		for (String columnName : updateColumns) {
			selectSQLJoin.add(columnName);
		}
	} else {
		for (String columnName : tableMeta.getAllColumns().keySet()) {
			selectSQLJoin.add(ColumnUtils.addEscape(columnName, getDbType()));
		}
	}
	// 生成SQL语句
	return selectSQLJoin.toString();
}

```
### (7). UpdateExecutor.afterImage
```
protected TableRecords afterImage(TableRecords beforeImage) throws SQLException {
	// 7.1 获得表的无数据
	TableMeta tmeta = getTableMeta();
	if (beforeImage == null || beforeImage.size() == 0) {
		return TableRecords.empty(getTableMeta());
	}
	
	// 7.1 构建镜像后的SQL语句
	// SELECT id, count FROM storage_tbl WHERE (id) in ( (?) )
	// 生成后的SQL是in(毕竟:可能是更新多条,自然是锁住多条,也会对N条数据进行镜像)
	String selectSQL = buildAfterImageSQL(tmeta, beforeImage);
	ResultSet rs = null;
	try (PreparedStatement pst = statementProxy.getConnection().prepareStatement(selectSQL)) {
		SqlGenerateUtils.setParamForPk(beforeImage.pkRows(), getTableMeta().getPrimaryKeyOnlyName(), pst);
		// 执行SQL语句,返回:ResultSet
		rs = pst.executeQuery();
		// 对ResultSet转换成业务模型对象:TableRecords
		return TableRecords.buildRecords(tmeta, rs);
	} finally {
		IOUtil.close(rs);
	}
}// end afterImage

private String buildAfterImageSQL(TableMeta tableMeta, TableRecords beforeImage) throws SQLException {
	StringBuilder prefix = new StringBuilder("SELECT ");
	// in
	String whereSql = SqlGenerateUtils.buildWhereConditionByPKs(tableMeta.getPrimaryKeyOnlyName(), beforeImage.pkRows().size(), getDbType());
	String suffix = " FROM " + getFromTableInSQL() + " WHERE " + whereSql;
	StringJoiner selectSQLJoiner = new StringJoiner(", ", prefix.toString(), suffix);
	if (ONLY_CARE_UPDATE_COLUMNS) {
		SQLUpdateRecognizer recognizer = (SQLUpdateRecognizer) sqlRecognizer;
		List<String> updateColumns = recognizer.getUpdateColumns();
		if (!containsPK(updateColumns)) {
			selectSQLJoiner.add(getColumnNamesInSQL(tableMeta.getEscapePkNameList(getDbType())));
		}
		for (String columnName : updateColumns) {
			selectSQLJoiner.add(columnName);
		}
	} else {
		for (String columnName : tableMeta.getAllColumns().keySet()) {
			selectSQLJoiner.add(ColumnUtils.addEscape(columnName, getDbType()));
		}
	}
	return selectSQLJoiner.toString();
} // end buildAfterImageSQL
```
### (8). BaseTransactionalExecutor.prepareUndoLog
```
protected void prepareUndoLog(TableRecords beforeImage, TableRecords afterImage) throws SQLException {
	// 8.1 检查
	if (beforeImage.getRows().isEmpty() && afterImage.getRows().isEmpty()) {
		return;
	}

	// 获得:ConnectionProxy
	ConnectionProxy connectionProxy = statementProxy.getConnectionProxy();
	
	// 如果SQL类型是:DELETE就返回beforeImage要锁住的记录
	// 如果SQL类型不是:DELETE,那就只能是:INSERT/UPDATE了,返回镜像之后的记录信息.
	TableRecords lockKeyRecords = sqlRecognizer.getSQLType() == SQLType.DELETE ? beforeImage : afterImage;
	
	
	// *******************************************************
	// 8.2 构建要锁住的多条记录
	// 规则: 表名:主键id,主键id...
	// 例如: storage_tbl:1
	// *******************************************************
	String lockKeys = buildLockKey(lockKeyRecords);
	
	// 8.3 把信息设置到:ConnectionProxy对象,ConnectionProxy是安全的,每次都是new出来的
	// 为什么放在ConnectionProxy,因为在ConnectionProxy.commit阶段,需要拿着这个信息去TC获得全局锁.
	connectionProxy.appendLockKey(lockKeys);

	// 8.4 把beforeImage和afterImage转换成业务模型:SQLUndoLog对象
	SQLUndoLog sqlUndoLog = buildUndoItem(beforeImage, afterImage);
	// 把SQLUndoLog设置到:ConnectionProxy.commit阶段,需要保证业务SQL和Undolong在同一个事务里的.
	connectionProxy.appendUndoLog(sqlUndoLog);
}
```
### (9). UpdateExecutor执行流程图解
!["UpdateExecutor执行图解"](/assets/seata/imgs/seata-UpdateExecutorSequence-Diagram.jpg)

### (10). 总结
> 1. ExecuteTemplate将请执行任务,委托给:UpdateExecutor.doExecute().      
> 2. 执行:UpdateExecutor.beforeImage(),注意:FOR UPDATE,并生成镜像.     
> 3. 执行:业务SQL语句.     
> 4. 执行:UpdateExecutor.afterImage().     
> 5. 调用:ConnectionProxy.commit()方法.    
> 6. 失败的情况下会调用:ConnectionProxy.rollback()方法.    