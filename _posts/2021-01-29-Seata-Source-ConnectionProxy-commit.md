---
layout: post
title: 'Seata 分支事务处理之ConnectionProxy提交/回滚事务详解(五)'
date: 2021-01-29
author: 李新
tags: Seata-AT源码
---

### (1).  概述
> 前一节,分析到在UpdateExecutor方法里,会调用:ConnectionProxy.commit()方法,没有进入到里面进行详解,这一小节,主要分析commit/rollback方法.    

### (2). ConnectionProxy.commit

```
public void commit() throws SQLException {
	try {
		LOCK_RETRY_POLICY.execute(() -> {
			// 2.1 委托给:doCommit()
			doCommit();
			return null;
		});
	} catch (SQLException e) {
		// 失败的情况下,进行:rollback,并向TC汇报失败.
		if (targetConnection != null && !getAutoCommit()) {
			rollback();
		}
		throw e;
	} catch (Exception e) {
		throw new SQLException(e);
	}
}// end commit


// 2.1 doCommit()
private void doCommit() throws SQLException {
	if (context.inGlobalTransaction()) {  // xid != null
		// 3. processGlobalTransactionCommit
		processGlobalTransactionCommit();
	} else if (context.isGlobalLockRequire()) { // isGlobalLockRequire == true
		processLocalCommitWithGlobalLocks();
	} else {  //直接commit
		targetConnection.commit();
	}
}// end doCommit
```
### (3). ConnectionProxy.processGlobalTransactionCommit
```
3. processGlobalTransactionCommit
private void processGlobalTransactionCommit() throws SQLException {
	try {
		// *************************************************************
		// 4. 注册分支事务
		// 在这里名称是叫做注册,实际还韵含着注册和获取全局锁的操作来着的.
		// *************************************************************
		register();
	} catch (TransactionException e) {
		// *************************************************************
		// 5. 注册分支事务失败,会封装异常,然后继续throw,压根就不会走下面的逻辑了.
		// *************************************************************
		recognizeLockKeyConflictException(e, context.buildLockKeys());
	}
		
	try {
		// 把undo_log刷盘
		UndoLogManagerFactory.getUndoLogManager(this.getDbType()).flushUndoLogs(this);
		
		// *************************************************************
		// 这里才是真正的提交
		// *************************************************************
		targetConnection.commit();
	} catch (Throwable ex) {
		LOGGER.error("process connectionProxy commit error: {}", ex.getMessage(), ex);
		
		// 向TC汇报异常
		// 6. ConnectionProxy.report
		report(false);
		throw new SQLException(ex);
	}
	
	if (IS_REPORT_SUCCESS_ENABLE) { // 默认值是:false
		// 当开启了汇报情况下,才会向TC汇报正常.
		// 这样设计的意思是:Seata(TC)默认认为是成功的,只有失败才会向TC汇报.
		// 6. ConnectionProxy.report
		report(true);
	}
	// 重置上下文.
	context.reset();
}
```
### (4). ConnectionProxy.register
```
private void register() throws TransactionException {
	if (!context.hasUndoLog() || context.getLockKeysBuffer().isEmpty()) {
		return;
	}
	
	// RM向TC注册分支事务.
	// 注册成功后,会产生分支事务ID
	Long branchId = DefaultResourceManager.get().branchRegister(BranchType.AT, getDataSourceProxy().getResourceId(),
		null, context.getXid(), null, context.buildLockKeys());
	// 设置分支事务ID	
	context.setBranchId(branchId);
}
```
### (5). ConnectionProxy.recognizeLockKeyConflictException
> 注册分支事务失败,封装异常信息,继续往外抛出异常.

```
private void recognizeLockKeyConflictException(TransactionException te, String lockKeys) throws SQLException {
	if (te.getCode() == TransactionExceptionCode.LockKeyConflict) { // 针对:TransactionExceptionCode.LockKeyConflict处理
		StringBuilder reasonBuilder = new StringBuilder("get global lock fail, xid:");
		reasonBuilder.append(context.getXid());
		if (StringUtils.isNotBlank(lockKeys)) {
			reasonBuilder.append(", lockKeys:").append(lockKeys);
		}
		throw new LockConflictException(reasonBuilder.toString());
	} else {
		throw new SQLException(te);
	}
}
```
### (6). ConnectionProxy.report
```
private void report(boolean commitDone) throws SQLException {
	// 不存在分支ID的情况下,跳过
	if (context.getBranchId() == null) {
		return;
	}
	
	// 最大重试次数为:5次
	int retry = REPORT_RETRY_COUNT; // 5
	while (retry > 0) {
		try {
			// RM向TC汇报情况
			DefaultResourceManager.get().branchReport(BranchType.AT, context.getXid(), context.getBranchId(),
				commitDone ? BranchStatus.PhaseOne_Done : BranchStatus.PhaseOne_Failed, null);
			return;
		} catch (Throwable ex) {
			// 有异常的情况下,重试,直到retry < 0
			LOGGER.error("Failed to report [" + context.getBranchId() + "/" + context.getXid() + "] commit done ["
				+ commitDone + "] Retry Countdown: " + retry);
			retry--;
			if (retry == 0) {
				throw new SQLException("Failed to report branch status " + commitDone, ex);
			}
		}
	}
}
```
### (7). 总结
> ConnectionProxy.commit方法的主要职责:  
> 1. 向TC获取全局锁(注册分支事事)   
> 2. 提交本地事务.   
> 3. 失败的情况下,进行rollback,并向TC汇报失败.   
> 4. 那什么时候?进行第二阶段的全局:commit/rollback呢?答案就在:RmNettyRemotingClient.registerProcessor方法里.   