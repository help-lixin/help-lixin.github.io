---
layout: post
title: 'Seata  TCC分支事务之TccActionInterceptor(一)'
date: 2021-01-29
author: 李新
tags: Seata-TCC源码
---

### (1). 概述
>  在前面的内容,剖析到:TccActionInterceptor的代码织入,仅在消费端生效(@LocalTCC除外).  

### (2). TccActionInterceptor.invoke
```
public Object invoke(final MethodInvocation invocation) throws Throwable {
	// 如果xid不存 或者 是否禁用全局事务  或者 是事为Saga分支
	if (!RootContext.inGlobalTransaction() || disable || RootContext.inSagaBranch()) { // false
		//not in transaction
		return invocation.proceed();
	}
	
	// 获得远程方法信息
	// io.seata.samples.tcc.transfer.action.FirstTccAction
	// @TwoPhaseBusinessAction(name = "firstTccAction", commitMethod = "commit", rollbackMethod = "rollback")
    // public boolean prepareMinus(
	//    BusinessActionContext businessActionContext,
    //    @BusinessActionContextParameter(paramName = "accountNo") String accountNo,
    //    @BusinessActionContextParameter(paramName = "amount") double amount);
	Method method = getActionInterfaceMethod(invocation);
	
	// 获得接口上的注解@TwoPhaseBusinessAction
	TwoPhaseBusinessAction businessAction = method.getAnnotation(TwoPhaseBusinessAction.class);
	//try method
	if (businessAction != null) { // 有注解@TwoPhaseBusinessAction才进该逻辑
		
		//save the xid
		// 获取ThreadLocal(RootContext)中的: 
		String xid = RootContext.getXID();
		//save the previous branchType
		// 获取ThreadLocal(RootContext)中的:BranchType
		BranchType previousBranchType = RootContext.getBranchType();
		
		// 如果不是TCC模型的话,设置为:TCC模型
		//if not TCC, bind TCC branchType
		if (BranchType.TCC != previousBranchType) {
			RootContext.bindBranchType(BranchType.TCC);
		}
		
		// 在此处要稍微思考:
		// 1. 分支事务什么时候注册? 
		//  答: 分支事务是在ActionInterceptorHandler.proceed方法里.
		// 2. 分支事务什么时候commit/rollback?
		//  答: commit和rollback都是由TC来驱动的.
		// 3. 为什么没有catch异常做处理?
		//  答: 所有的分支事务,被全局事务给包裹着,只要任何分支事务抛出了异常,会被全局事务给拦截,全局事务会告之TC,TC会驱动进行rollback.
		
		try {
			Object[] methodArgs = invocation.getArguments();
			//Handler the TCC Aspect
			// ************************************
			// 3.1 获得目标对象和方法
			// 3.2 actionInterceptorHandler.proceed()
			// ActionInterceptorHandler类的内容,我留到下一节再剖析.
			// ************************************
			Map<String, Object> ret = actionInterceptorHandler.proceed(
			                                 method, 
											 methodArgs, 
											 xid, 
											 businessAction,
											 // ************************************
											 // 3.1 获得目标对象和方法
											 // ************************************
											 invocation::proceed);
			//return the final result
			return ret.get(Constants.TCC_METHOD_RESULT);
		}
		finally {
			// 如果不是TCC模型,还原成原来的模型(AT/Saga)
			//if not TCC, unbind branchType
			if (BranchType.TCC != previousBranchType) {
				RootContext.unbindBranchType();
			}
		}
	}
	return invocation.proceed();
}// end invoke
```
### (3). 总结
> TccActionInterceptor的主要职责: 
> 拦截消费者(Client)的调用,拦截方法上指定的注解(@TwoPhaseBusinessAction).    