---
layout: post
title: 'Seata 全局事务处理之TransactionManager(八)'
date: 2021-01-29
author: 李新
tags: Seata源码
---

### (1). TransactionManager
> 在这一小节,我主要分析:DefaultTransactionManager,因为它是:TransactionManager的实现类. 

### (2). DefaultTransactionManager
```
public class DefaultTransactionManager implements TransactionManager {

	// begin
    @Override
    public String begin(String applicationId, String transactionServiceGroup, String name, int timeout)
        throws TransactionException {
			
	    // 创建:GlobalBeginRequest它属于:AbstractMessage的子类
		// 在前面有分析过:AbstractMessage是Seata在Netty上基于ProtoBuf自定义的协议栈
        GlobalBeginRequest request = new GlobalBeginRequest();
		// 如果注解有指定名称就用注解上的名称,否则,就是方法签名了
        request.setTransactionName(name);
        request.setTimeout(timeout);
		
		// syncCall == this.syncCall
		// 调用:TmNettyRemotingClient.getInstance().sendSyncRequest(request)
        GlobalBeginResponse response = (GlobalBeginResponse) syncCall(request);
        if (response.getResultCode() == ResultCode.Failed) {
            throw new TmTransactionException(TransactionExceptionCode.BeginFailed, response.getMsg());
        }
        return response.getXid();
    }

	// commit
    @Override
    public GlobalStatus commit(String xid) throws TransactionException {
		// 创建:GlobalCommitRequest它属于:AbstractMessage的子类
		// 在前面有分析过:AbstractMessage是Seata在Netty上基于ProtoBuf自定义的协议栈
        GlobalCommitRequest globalCommit = new GlobalCommitRequest();
        globalCommit.setXid(xid);
		
		// syncCall == this.syncCall
		// 调用:TmNettyRemotingClient.getInstance().sendSyncRequest(request)
        GlobalCommitResponse response = (GlobalCommitResponse) syncCall(globalCommit);
        return response.getGlobalStatus();
    }

	// rollback
    @Override
    public GlobalStatus rollback(String xid) throws TransactionException {
		// 创建:GlobalRollbackRequest它属于:AbstractMessage的子类
		// 在前面有分析过:AbstractMessage是Seata在Netty上基于ProtoBuf自定义的协议栈
        GlobalRollbackRequest globalRollback = new GlobalRollbackRequest();
        globalRollback.setXid(xid);
		
		// syncCall == this.syncCall
		// 调用:TmNettyRemotingClient.getInstance().sendSyncRequest(request)
        GlobalRollbackResponse response = (GlobalRollbackResponse) syncCall(globalRollback);
        return response.getGlobalStatus();
    }

	// 向TC获得XID的状态
    @Override
    public GlobalStatus getStatus(String xid) throws TransactionException {
		// 创建:GlobalStatusRequest它属于:AbstractMessage的子类
		// 在前面有分析过:AbstractMessage是Seata在Netty上基于ProtoBuf自定义的协议栈
        GlobalStatusRequest queryGlobalStatus = new GlobalStatusRequest();
        queryGlobalStatus.setXid(xid);
		
		// syncCall == this.syncCall
		// 调用:TmNettyRemotingClient.getInstance().sendSyncRequest(request)
        GlobalStatusResponse response = (GlobalStatusResponse) syncCall(queryGlobalStatus);
        return response.getGlobalStatus();
    }

	// 向TC汇报状态
    @Override
    public GlobalStatus globalReport(String xid, GlobalStatus globalStatus) throws TransactionException {
		// 创建:GlobalReportRequest它属于:AbstractMessage的子类
		// 在前面有分析过:AbstractMessage是Seata在Netty上基于ProtoBuf自定义的协议栈
        GlobalReportRequest globalReport = new GlobalReportRequest();
        globalReport.setXid(xid);
        globalReport.setGlobalStatus(globalStatus);
		
		// syncCall == this.syncCall
		// 调用:TmNettyRemotingClient.getInstance().sendSyncRequest(request)
        GlobalReportResponse response = (GlobalReportResponse) syncCall(globalReport);
        return response.getGlobalStatus();
    }

	// ****************************************************************
	// 通过:TmNettyRemotingClient进行同步调用
	// ****************************************************************
    private AbstractTransactionResponse syncCall(AbstractTransactionRequest request) throws TransactionException {
        try {
			// **********************************************************************
			// TmNettyRemotingClient.getInstance().sendSyncRequest我在前面已经分析过了
			// **********************************************************************
            return (AbstractTransactionResponse) TmNettyRemotingClient.getInstance().sendSyncRequest(request);
        } catch (TimeoutException toe) {
            throw new TmTransactionException(TransactionExceptionCode.IO, "RPC timeout", toe);
        }
    }
}
```
### (3). 整个链路调用图解
!["GlobalTransactionalInterceptor链路调用图解"](/assets/seata/imgs/seata-GlobalTransactionalInterceptor-Sequence.jpg)

### (4). 总结
> GlobalTransactionalInterceptor的职责:  
> 1. 拦截注解(AOP).
> 2. 向TC发送指令(begin),产生xid.   
> 3. 执行业务逻辑代码.    
> 4. 向TC发送指令(commit/rollback).   
> 5. 留一个疑问?分析了这么久,都只是分析到全局事务的处理,那么分支事务是如何做的呢?哈哈哈,且跟着我继续往下走.  