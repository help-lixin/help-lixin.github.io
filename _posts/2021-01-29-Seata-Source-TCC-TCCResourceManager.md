---
layout: post
title: 'Seata  TCC分支事务之TCCResourceManager提交或回滚(三)'
date: 2021-01-29
author: 李新
tags: Seata-TCC源码
---

### (1). 分支事务是何时commit和rollback
> 和AT模式是一样的,分支事务都是要等待TC大体流程如下:   
> 1. RmBranchCommitProcessor接受TC发送过来的事件.       
> 2. DefaultRMHandler.onRequest对请求进行转发.          
> 3. DefaultRMHandler.handle对请求进行处理.      
> 4. 最终:TCCResourceManager.branchCommit/branchRollback进行处理.  

### (2). TCCResourceManager.branchCommit

```
public BranchStatus branchCommit(
		BranchType branchType, 
		String xid, 
		long branchId, 
		String resourceId,
		String applicationData) throws TransactionException {
						
	// @TwoPhaseBusinessAction(name = "firstTccAction", commitMethod = "commit", rollbackMethod = "rollback")
	// 在分支事务,进行注册的时候,就在保存资源ID(firstTccAction)
	// ActionInterceptorHandler.doTccActionLogStore
	TCCResource tccResource = (TCCResource)tccResourceCache.get(resourceId);
	
	// 如果是这样,是否代表着RM是有状态的?
	if (tccResource == null) {
		throw new ShouldNeverHappenException(String.format("TCC resource is not exist, resourceId: %s", resourceId));
	}
	
	// Bean是Spring创建的
	Object targetTCCBean = tccResource.getTargetBean();
	// commit
	Method commitMethod = tccResource.getCommitMethod();
	// 检查下Bean和对应的方法是否存在
	if (targetTCCBean == null || commitMethod == null) {
		throw new ShouldNeverHappenException(String.format("TCC resource is not available, resourceId: %s", resourceId));
	}
	
	try {
		// 在 ActionInterceptorHandler.doTccActionLogStore会创建:BusinessActionContext
		// 把数据转换成了JSON,并提交给了TC.
		在这阶段,TC会把数据传过来,需要把JSON数据反序列化成:BusinessActionContext
		//BusinessActionContext
		BusinessActionContext businessActionContext = getBusinessActionContext(xid, branchId, resourceId,
			applicationData);
			
		// ****************************************************************************	
		// 	调用业务方法
		// ****************************************************************************	
		Object ret = commitMethod.invoke(targetTCCBean, businessActionContext);
		LOGGER.info("TCC resource commit result : {}, xid: {}, branchId: {}, resourceId: {}", ret, xid, branchId, resourceId);
		
		// *****************************************************************
		// TCC看来是比较强性的规定:commit的返回类型了.
		// *****************************************************************
		// 1. 返回结果(ret)如果为null,则直接告诉TC:BranchStatus.PhaseTwo_Committed.  
		// 2. 如果ret == TwoPhaseResult,获取:TwoPhaseResult.success,根据success,决定返回:PhaseTwo_Committed/PhaseTwo_CommitFailed_Retryable.  
		// 3. 把ret强制转换成Booleean,再决定返回:PhaseTwo_Committed/PhaseTwo_CommitFailed_Retryable.
		// 4. 总结:TCC的commit的方法签名,返回值反正建议是:Boolean.
		boolean result;
		if (ret != null) {
			if (ret instanceof TwoPhaseResult) {
				result = ((TwoPhaseResult)ret).isSuccess();
			} else {
				result = (boolean)ret;
			}
		} else {
			result = true;
		}
		// result == true ==> BranchStatus.PhaseTwo_Committed 
		// result == false ==> BranchStatus.PhaseTwo_CommitFailed_Retryable
		return result ? BranchStatus.PhaseTwo_Committed : BranchStatus.PhaseTwo_CommitFailed_Retryable;
	} catch (Throwable t) {
		// **********************************************************
		// 抛出异常时,通知重试
		// **********************************************************
		String msg = String.format("commit TCC resource error, resourceId: %s, xid: %s.", resourceId, xid);
		LOGGER.error(msg, t);
		return BranchStatus.PhaseTwo_CommitFailed_Retryable;
	}
}
```
### (3). TCCResourceManager.branchRollback
```
public BranchStatus branchRollback(
	BranchType branchType, 
	String xid, 
	long branchId, 
	String resourceId,
	String applicationData) throws TransactionException {
		
	TCCResource tccResource = (TCCResource)tccResourceCache.get(resourceId);
	if (tccResource == null) {
		throw new ShouldNeverHappenException(String.format("TCC resource is not exist, resourceId: %s", resourceId));
	}
	Object targetTCCBean = tccResource.getTargetBean();
	Method rollbackMethod = tccResource.getRollbackMethod();
	if (targetTCCBean == null || rollbackMethod == null) {
		throw new ShouldNeverHappenException(String.format("TCC resource is not available, resourceId: %s", resourceId));
	}
	try {
		//BusinessActionContext
		BusinessActionContext businessActionContext = getBusinessActionContext(xid, branchId, resourceId,
			applicationData);
		// *******************************************************************
		// 调用业务方法
		// *******************************************************************
		Object ret = rollbackMethod.invoke(targetTCCBean, businessActionContext);
		LOGGER.info("TCC resource rollback result : {}, xid: {}, branchId: {}, resourceId: {}", ret, xid, branchId, resourceId);
		boolean result;
		if (ret != null) {
			if (ret instanceof TwoPhaseResult) {
				result = ((TwoPhaseResult)ret).isSuccess();
			} else {
				result = (boolean)ret;
			}
		} else {
			result = true;
		}
		// 返回:BranchStatus.PhaseTwo_Rollbacked/PhaseTwo_RollbackFailed_Retryable
		return result ? BranchStatus.PhaseTwo_Rollbacked : BranchStatus.PhaseTwo_RollbackFailed_Retryable;
	} catch (Throwable t) {
		String msg = String.format("rollback TCC resource error, resourceId: %s, xid: %s.", resourceId, xid);
		LOGGER.error(msg, t);
		return BranchStatus.PhaseTwo_RollbackFailed_Retryable;
	}
}
```
### (4). 总结
> 1. @GlobalTransactional向TC注册全局事务.并通过网络透传xid.   
> 2. RM绑定远程xid到本地线程,当遇到(@TwoPhaseBusinessAction/@LocalTCC)时,会向TC注册分支事务,并调用业务代码的try方法.       
> 3. @GlobalTransactional决定着整个全局事务的commit和rollback.     
> 4. 如果是commit,则,由TC通知所有的参与者(RM),调用commit.     
> 5. 如果是rollback,则,由TC通知所有的参与者(RM),调用rollback.   