---
layout: post
title: 'Seata  全局事务处理之GlobalTransaction(七)'
date: 2021-01-29
author: 李新
tags: Seata源码
---

### (1). 概述
> 前面分析到:TransactionalTemplate会把请求委派给:GlobalTransaction处理.
> 在这里我将分析:DefaultGlobalTransaction,因为它是:GlobalTransaction的实现类.   

### (2). GlobalTransaction类详解
> 我在这里只剖析:TransactionalTemplate用到的三个方法(begin/commit/rollback).  

### (3). 先看一下DefaultGlobalTransaction的构造器和相关的属性
```
private static final int DEFAULT_GLOBAL_TX_TIMEOUT = 60000;
private static final String DEFAULT_GLOBAL_TX_NAME = "default";

// 依赖: TransactionManager
private TransactionManager transactionManager;

private String xid;
private GlobalStatus status;
private GlobalTransactionRole role;

// commit最大重试次数
private static final int COMMIT_RETRY_COUNT = ConfigurationFactory.getInstance().getInt(
	ConfigurationKeys.CLIENT_TM_COMMIT_RETRY_COUNT, DEFAULT_TM_COMMIT_RETRY_COUNT);
	
// rollback最大重试次数
private static final int ROLLBACK_RETRY_COUNT = ConfigurationFactory.getInstance().getInt(
	ConfigurationKeys.CLIENT_TM_ROLLBACK_RETRY_COUNT, DEFAULT_TM_ROLLBACK_RETRY_COUNT);

// 构造器
DefaultGlobalTransaction() {
	// xid为空
	// 状态为:UnKnown(未开始)
	// 角色为:GlobalTransactionRole.Launcher
	this(null, GlobalStatus.UnKnown, GlobalTransactionRole.Launcher);
}

// 构造器
DefaultGlobalTransaction(String xid, GlobalStatus status, GlobalTransactionRole role) {
	// 通过SPI获得:TransactionManager
	this.transactionManager = TransactionManagerHolder.get();
	this.xid = xid;
	this.status = status;
	this.role = role;
}
```
### (4). DefaultGlobalTransaction.begin
```
@Override
public void begin() throws TransactionException {
	begin(DEFAULT_GLOBAL_TX_TIMEOUT);
}

@Override
public void begin(int timeout) throws TransactionException {
	begin(timeout, DEFAULT_GLOBAL_TX_NAME);
}

@Override
public void begin(int timeout, String name) throws TransactionException {
	if (role != GlobalTransactionRole.Launcher) {
		// 当前的角色为参与者是(Participant) xid是必须要先存在的
		// if (xid == null) {
        //     throw new IllegalStateException();
        // }
		assertXIDNotNull();  // 若xid为空,抛出异常:IllegalStateException
		if (LOGGER.isDebugEnabled()) {
			LOGGER.debug("Ignore Begin(): just involved in global transaction [{}]", xid);
		}
		// 直接返回了,代表XID不用再注册了
		return;
	}
	
	
	// if (xid != null) {
    //   throw new IllegalStateException();
    // }
	assertXIDNull();  //若xid不为空(xid已经存在了),则抛出异常:IllegalStateException
	
	// 从ThreadLocal获得xid(TX_XID)
	String currentXid = RootContext.getXID();
	if (currentXid != null) {  // 存在,抛出异常
		throw new IllegalStateException("Global transaction already exists," +
			" can't begin a new global transaction, currentXid = " + currentXid);
	}
	// 上面的逻辑是:在角色为:GlobalTransactionRole.Launcher的情况下,xid必须要为空.否则,会进行检查
	
	// *******************************************************
	// 委托给了:DefaultTransactionManager进行处理
	// 我在这里不详解,留一节在后面详解.
	// *******************************************************
	xid = transactionManager.begin(null, null, name, timeout);
	
	// 设置状态为:Begin
	status = GlobalStatus.Begin;
	
	// 设置xid到ThreadLocal里
	RootContext.bind(xid);
	
	// 打印日志
	if (LOGGER.isInfoEnabled()) {
		LOGGER.info("Begin new global transaction [{}]", xid);
	}
}
```
### (5). DefaultGlobalTransaction.commit
```
public void commit() throws TransactionException {
	
	if (role == GlobalTransactionRole.Participant) {
		// Participant has no responsibility of committing
		if (LOGGER.isDebugEnabled()) {
			LOGGER.debug("Ignore Commit(): just involved in global transaction [{}]", xid);
		}
		return;
	}
	
	// if (xid == null) {
    //     throw new IllegalStateException();
	// }
	assertXIDNotNull();  // commit阶段了,xid是上不能为空的了
	
	
	// 获得配置参数(commit的最大重试次数):
	// client.tm.commitRetryCount = 5
	int retry = COMMIT_RETRY_COUNT <= 0 ? DEFAULT_TM_COMMIT_RETRY_COUNT : COMMIT_RETRY_COUNT;
	try {
		// commit
		while (retry > 0) {
			try {
				// *******************************************************
				// 委托给了:DefaultTransactionManager进行处理
				// 我在这里不详解,留一节在后面详解.
				// *******************************************************
				status = transactionManager.commit(xid);
				break;
			} catch (Throwable ex) {
				LOGGER.error("Failed to report global commit [{}],Retry Countdown: {}, reason: {}", this.getXid(), retry, ex.getMessage());
				retry--;
				if (retry == 0) {
					throw new TransactionException("Failed to report global commit", ex);
				}
			}
		}
	} finally {
		if (xid.equals(RootContext.getXID())) {
			suspend();
		}
	}
	if (LOGGER.isInfoEnabled()) {
		LOGGER.info("[{}] commit status: {}", xid, status);
	}
}
```
### (6). DefaultGlobalTransaction.rollback
```
public void rollback() throws TransactionException {
	
	if (role == GlobalTransactionRole.Participant) {
		// Participant has no responsibility of rollback
		if (LOGGER.isDebugEnabled()) {
			LOGGER.debug("Ignore Rollback(): just involved in global transaction [{}]", xid);
		}
		return;
	}
	
	// if (xid == null) {
    //        throw new IllegalStateException();
    // }
	assertXIDNotNull(); // rollback阶段,xid也不能为空

	// 读取参数(rollback最大重试次数)
	// client.tm.rollbackRetryCount = 5 
	int retry = ROLLBACK_RETRY_COUNT <= 0 ? DEFAULT_TM_ROLLBACK_RETRY_COUNT : ROLLBACK_RETRY_COUNT;
	try {
		while (retry > 0) {
			try {
				// *******************************************************
				// 委托给了:DefaultTransactionManager进行处理
				// 我在这里不详解,留一节在后面详解.
				// *******************************************************
				status = transactionManager.rollback(xid);
				break;
			} catch (Throwable ex) {
				LOGGER.error("Failed to report global rollback [{}],Retry Countdown: {}, reason: {}", this.getXid(), retry, ex.getMessage());
				retry--;
				if (retry == 0) {
					throw new TransactionException("Failed to report global rollback", ex);
				}
			}
		}
	} finally {
		if (xid.equals(RootContext.getXID())) {
			suspend();
		}
	}
	if (LOGGER.isInfoEnabled()) {
		LOGGER.info("[{}] rollback status: {}", xid, status);
	}
}
```
### (7). 总结
> GlobalTransaction接受(begin/commit/rollback)请求,进行参数的校验,然后委托给:TransactionManager进行处理.   