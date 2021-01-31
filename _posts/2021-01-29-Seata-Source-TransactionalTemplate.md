---
layout: post
title: 'Seata TransactionalTemplate(六)'
date: 2021-01-29
author: 李新
tags: Seata源码
---

### (1). 概述
> 前面了解到GlobalTransactionalInterceptor,它主要负责拦截@GlobalTransactional(@GlobalLock).并把请求委派给:TransactionalTemplate.  

### (2). TransactionalTemplate.execute
```
public Object execute(TransactionalExecutor business) throws Throwable {
	// 把@GlobalTransactional转换成:业务对象(TransactionInfo)   
	// 1. Get transactionInfo
	TransactionInfo txInfo = business.getTransactionInfo();

	if (txInfo == null) {
		throw new ShouldNeverHappenException("transactionInfo does not exist");
	}
	
	// 1. 从ThreadLocal中获得TX_XID,赋给临时变量:xid
	// 2. 如果xid为空,则直接返回:GlobalTransaction tx = null.
	// 3. 如果xid不为空,则创建:new DefaultGlobalTransaction(xid, GlobalStatus.Begin, GlobalTransactionRole.Participant);
	//    有xid的情况下,代表这是一个:参与者.  
	// 1.1 Get current transaction, if not null, the tx role is 'GlobalTransactionRole.Participant'.
	GlobalTransaction tx = GlobalTransactionContext.getCurrent();

	
	// 获得事务的传播行为(默认是:Propagation.REQUIRED)
	// 1.2 Handle the transaction propagation.
	Propagation propagation = txInfo.getPropagation();
	
	// 挂起资源(越看越像Spring里的事务代码了)
	SuspendedResourcesHolder suspendedResourcesHolder = null;
	try {
		
		switch (propagation) {  // 判断传播行为
			case NOT_SUPPORTED:
				// If transaction is existing, suspend it.
				// 这种情况,代表(GlobalTransaction)事务已经存在了,因为判断了不允许为空.
				if (existingTransaction(tx)) {   // tx != null
					suspendedResourcesHolder = tx.suspend();
				}
				// Execute without transaction and return.
				return business.execute();
			case REQUIRES_NEW:
				// 这种情况,代表(GlobalTransaction)事务已经存在了,因为判断了不允许为空.
				// If transaction is existing, suspend it, and then begin new transaction.
				if (existingTransaction(tx)) {  // tx != null
					// 先挂起,后再创建一新的tx
					suspendedResourcesHolder = tx.suspend();
					tx = GlobalTransactionContext.createNew();
				}
				// Continue and execute with new transaction
				break;
			case SUPPORTS:
				// 这种情况,代表(GlobalTransaction)事务不允许存在
				// If transaction is not existing, execute without transaction.
				if (notExistingTransaction(tx)) { // tx == null
					return business.execute();
				}
				// Continue and execute with new transaction
				break;
			case REQUIRED:
				// If current transaction is existing, execute with current transaction,
				// else continue and execute with new transaction.
				break;
			case NEVER:
				// 这种情况,(GlobalTransaction)事务若存在,则抛出异常(TransactionException)
				// 若不存在,则执行业务逻辑.  
				// If transaction is existing, throw exception.
				if (existingTransaction(tx)) {
					throw new TransactionException(
						String.format("Existing transaction found for transaction marked with propagation 'never', xid = %s"
								, tx.getXid()));
				} else {
					// Execute without transaction and return.
					return business.execute();
				}
			case MANDATORY:
				// 这种情况,代表(GlobalTransaction)事务不存在,则抛出异常:TransactionException
				// If transaction is not existing, throw exception.
				if (notExistingTransaction(tx)) {
					throw new TransactionException("No existing transaction found for transaction marked with propagation 'mandatory'");
				}
				// Continue and execute with current transaction.
				break;
			default:
				throw new TransactionException("Not Supported Propagation:" + propagation);
		}

		// ***********************************************************************
		// 如果tx为空,则创建一个新的tx,它的角色为:GlobalTransactionRole.Launcher  
		// ***********************************************************************
		// 1.3 If null, create new transaction with role 'GlobalTransactionRole.Launcher'.
		if (tx == null) {
			// 此时全局事务xid还不存在的.
			// 创建:DefaultGlobalTransaction 内部持有一个对象:DefaultTransactionManager
			// 通过SPI获取到的DefaultTransactionManager
			// DefaultGlobalTransaction() {
			// this(null, GlobalStatus.UnKnown, GlobalTransactionRole.Launcher);
			// }
			tx = GlobalTransactionContext.createNew();
		}

		//  把@GlobalTransactional转换成:业务对象(TransactionInfo)   
		// 1. 把业务对象(TransactionInfo)的信息设置到ThreadLocal(GlobalLockConfigHolder)对象里.
		// set current tx config to holder
		GlobalLockConfig previousConfig = replaceGlobalLockConfig(txInfo);

		try {
			// 这代码的注解很全面.
			// 如果角色是:GlobalTransactionRole.Launcher,发送信息到TC(事务协调器)
			// 2. If the tx role is 'GlobalTransactionRole.Launcher', send the request of beginTransaction to TC,
			//    else do nothing. Of course, the hooks will still be triggered.
			// 3. 查看(TransactionalTemplate.beginTransaction)
			beginTransaction(txInfo, tx);
			
			Object rs;
			try {
				// ***************************************************************
				// 执行业务逻辑
				// ***************************************************************
				// Do Your Business
				rs = business.execute();
			} catch (Throwable ex) {
				// ***************************************************************
				// 抛出异常时的处理
				// ***************************************************************
				// 3. The needed business exception to rollback.
				completeTransactionAfterThrowing(txInfo, tx, ex);
				throw ex;
			}

			// ***************************************************************
			// 正确时的提交
			// 4. everything is fine, commit.
			// ***************************************************************
			commitTransaction(tx);

			return rs;
		} finally {
			// ***************************************************************
			// 清理现场 
			// ***************************************************************
			//5. clear
			// 清除ThreaLocal上的GlobalLockConfig
			resumeGlobalLockConfig(previousConfig);
			// 执行钩子函数:TransactionHook.afterCompletion方法
			triggerAfterCompletion();
			// 清除ThreadLocal上的TransactionHook
			cleanUp();
		}
	} finally {
		// 把挂起的资源拿出来,重新设置到ThreadLocal.
		// If the transaction is suspended, resume it.
		if (suspendedResourcesHolder != null) {
			tx.resume(suspendedResourcesHolder);
		}
	}
}
```
### (3). TransactionalTemplate.beginTransaction
> 开始全局事务.    
> 在执行全局事务之前,先回调相应的钩子函数.    
> 在执行全局事务之后,先回调相应的钩子函数.     

```
// 2. triggerBeforeBegin
private void triggerBeforeBegin() {
	for (TransactionHook hook : getCurrentHooks()) {
		try {
			hook.beforeBegin();
		} catch (Exception e) { // 人家考虑还是挺周到的,遍历钩子函数并执行若是出错,也不能影响全局,仅仅打印error出日志
			LOGGER.error("Failed execute beforeBegin in hook {}", e.getMessage(), e);
		}
	}
} // end triggerBeforeBegin

// 1. beginTransaction
private void beginTransaction(TransactionInfo txInfo, GlobalTransaction tx) throws TransactionalExecutor.ExecutionException {
	try {
		// 注册全局事务前,执行:TransactionHook.beforeBegin函数
		triggerBeforeBegin();
		
		// *****************************************************
		// 执行全局事务的注册
		// 在这里我不详解,后面我留一节,专门讲解这部份
		// GlobalTransaction.begin
		// *****************************************************
		tx.begin(txInfo.getTimeOut(), txInfo.getName());
		
		// 注册全局事务后,执行:TransactionHook.afterBegin函数
		triggerAfterBegin();
	} catch (TransactionException txe) {
		// 注册失败,则抛出异常(TransactionalExecutor.ExecutionException)
		throw new TransactionalExecutor.ExecutionException(tx, txe,
			TransactionalExecutor.Code.BeginFailure);
	}
} // end beginTransaction


private void triggerAfterBegin() {
	for (TransactionHook hook : getCurrentHooks()) {
		try {
			hook.afterBegin();
		} catch (Exception e) {
			LOGGER.error("Failed execute afterBegin in hook {}", e.getMessage(), e);
		}
	}
} // end triggerAfterBegin
```
### (4). TransactionalTemplate.completeTransactionAfterThrowing
> 进入方法的前提是,调用了业务方法时出现了异常.  
> 如果抛出的异常(@GlobalTransactional),用户有自定义异常处理,则,把异常交给用户自己处理.  
> 如果抛出的异常,用户没有自定义处理,Seta默认还是会再commit一次.   

```
private void completeTransactionAfterThrowing(TransactionInfo txInfo, GlobalTransaction tx, Throwable originalException) throws TransactionalExecutor.ExecutionException {
	//roll back
	// 1. 获得注解@GlobalTransactional(rollbackFor=xxx,rollbackForClassName=xxx,noRollbackFor=xxx,noRollbackForClassName=xxxx)
	// 2. 判断异常是否指定的异常
	if (txInfo != null && txInfo.rollbackOn(originalException)) {
		try {
			// 回调到自定义的异常处理类上.
			rollbackTransaction(tx, originalException);
		} catch (TransactionException txe) {
			// 回调失败的情况下,抛出:TransactionalExecutor.ExecutionException
			// 异常信息为:TransactionalExecutor.Code.RollbackFailure
			// Failed to rollback
			throw new TransactionalExecutor.ExecutionException(tx, txe,
					TransactionalExecutor.Code.RollbackFailure, originalException);
		}
	} else {
		// 继续commit
		// not roll back on this exception, so commit
		commitTransaction(tx);
	}
} // end completeTransactionAfterThrowing


private void rollbackTransaction(GlobalTransaction tx, Throwable originalException) throws TransactionException, TransactionalExecutor.ExecutionException {
	// TransactionHook.beforeRollback
	triggerBeforeRollback();
	
	// *******************************************************************
	// 在这里,我先不讨论,会留一节,专门讨论这个类:GlobalTransaction
	// *******************************************************************
	// GlobalTransaction.rollback()
	tx.rollback();
	
	// TransactionHook.afterRollback
	triggerAfterRollback();
	// 3.1 Successfully rolled back
	throw new TransactionalExecutor.ExecutionException(tx, GlobalStatus.RollbackRetrying.equals(tx.getLocalStatus())
		? TransactionalExecutor.Code.RollbackRetrying : TransactionalExecutor.Code.RollbackDone, originalException);
} // end rollbackTransaction
```
### (5). TransactionalTemplate.commitTransaction
```
// 执行业务逻辑正常的情况下.执行:commitTransaction
private void commitTransaction(GlobalTransaction tx) throws TransactionalExecutor.ExecutionException {
	try {
		// 执行:TransactionHook.beforeCommit
		triggerBeforeCommit();
		
		// *********************************************************************
		// 这一部份的内容,我留到后面,再详细
		// 执行GlobalTransaction.commit
		// *********************************************************************
		tx.commit();
		
		// 执行:TransactionHook.afterCommit
		triggerAfterCommit();
	} catch (TransactionException txe) {
		// 4.1 Failed to commit
		throw new TransactionalExecutor.ExecutionException(tx, txe,
			TransactionalExecutor.Code.CommitFailure);
	}
} // end commitTransaction
```

### (6). 总结
> TransactionalTemplate的职责如下:  
> 1. 向TC事务协调器注册全局事务(执行begin).  
> 2. 调用业务逻辑. 
> 3. 失败时,rollback全局事务.   
> 4. 成功时,commit全局事务.   
> 5. 用户可以在ThreadLocal中注册钩子函数(TransactionHook),Seta会在回调相应的函数.    
> 6. begin/commit/rollback会委托给:GlobalTransaction去执行.  
> 7. 下一节,主要分析:GlobalTransaction. 