---
layout: post
title: 'Seata  全局事务处理之GlobalTransactionalInterceptor(四)'
date: 2021-01-29
author: 李新
tags: Seata-AT源码
---

### (1). 先聊一下Spring
> GlobalTransactionalInterceptor在什么时候创建的?  
> GlobalTransactionalInterceptor是在GlobalTransactionScanner.wrapIfNecessary方法构建的.   
> GlobalTransactionScanner.wrapIfNecessary的业务逻辑功能请参考该链接[](/2021/01/29/Seata-Source-GlobalTransactionScanner.html)  
> GlobalTransactionScanner会判断Bean上是否有相应的注解,如果有注解,就会对Bean织入业务逻辑(我们简称AOP),而织入的业务逻辑就在:GlobalTransactionalInterceptor,Spring要求织入的类需要实现:org.aopalliance.intercept.MethodInterceptor   

### (2). GlobalTransactionalInterceptor构造器
> 构造器主要是获得配置中的参数,并记录在实例的属性上.部份属性是什么意思,请参考官网.  

```
// domain
private static final FailureHandler DEFAULT_FAIL_HANDLER = new DefaultFailureHandlerImpl();
private static int defaultGlobalTransactionTimeout = 0;

// **********************************************************************
// 需要关注下这几个属性
// **********************************************************************
private static final FailureHandler DEFAULT_FAIL_HANDLER = new DefaultFailureHandlerImpl();
private final TransactionalTemplate transactionalTemplate = new TransactionalTemplate();
private final GlobalLockTemplate globalLockTemplate = new GlobalLockTemplate();
private final FailureHandler failureHandler;


// methods
public GlobalTransactionalInterceptor(FailureHandler failureHandler) {
	// 如果failureHandler为空,则指定默认的:FailureHandler
	this.failureHandler = failureHandler == null ? DEFAULT_FAIL_HANDLER : failureHandler;
	
	// service.disableGlobalTransaction = false
	this.disable = ConfigurationFactory.getInstance().getBoolean(ConfigurationKeys.DISABLE_GLOBAL_TRANSACTION,
		DEFAULT_DISABLE_GLOBAL_TRANSACTION);
		
    // client.tm.degradeCheck = false
	degradeCheck = ConfigurationFactory.getInstance().getBoolean(ConfigurationKeys.CLIENT_DEGRADE_CHECK,
		DEFAULT_TM_DEGRADE_CHECK);
	if (degradeCheck) {
		ConfigurationCache.addConfigListener(ConfigurationKeys.CLIENT_DEGRADE_CHECK, this);
		degradeCheckPeriod = ConfigurationFactory.getInstance().getInt(
			ConfigurationKeys.CLIENT_DEGRADE_CHECK_PERIOD, DEFAULT_TM_DEGRADE_CHECK_PERIOD);
		degradeCheckAllowTimes = ConfigurationFactory.getInstance().getInt(
			ConfigurationKeys.CLIENT_DEGRADE_CHECK_ALLOW_TIMES, DEFAULT_TM_DEGRADE_CHECK_ALLOW_TIMES);
		EVENT_BUS.register(this);
		if (degradeCheckPeriod > 0 && degradeCheckAllowTimes > 0) {
			startDegradeCheck();
		}
	}
	this.initDefaultGlobalTransactionTimeout();
} // end 


private void initDefaultGlobalTransactionTimeout() {
	// true
	if (GlobalTransactionalInterceptor.defaultGlobalTransactionTimeout <= 0) {
		int defaultGlobalTransactionTimeout;
		try {
			// 60000
			defaultGlobalTransactionTimeout = ConfigurationFactory.getInstance().getInt(
					ConfigurationKeys.DEFAULT_GLOBAL_TRANSACTION_TIMEOUT, DEFAULT_GLOBAL_TRANSACTION_TIMEOUT);
		} catch (Exception e) {
			LOGGER.error("Illegal global transaction timeout value: " + e.getMessage());
			defaultGlobalTransactionTimeout = DEFAULT_GLOBAL_TRANSACTION_TIMEOUT;
		}
		if (defaultGlobalTransactionTimeout <= 0) {
			LOGGER.warn("Global transaction timeout value '{}' is illegal, and has been reset to the default value '{}'",
					defaultGlobalTransactionTimeout, DEFAULT_GLOBAL_TRANSACTION_TIMEOUT);
			defaultGlobalTransactionTimeout = DEFAULT_GLOBAL_TRANSACTION_TIMEOUT;
		}
		GlobalTransactionalInterceptor.defaultGlobalTransactionTimeout = defaultGlobalTransactionTimeout;
	}
} //end initDefaultGlobalTransactionTimeout
```

### (3). GlobalTransactionalInterceptor.invoke
> 注意:只有有标注于相应的注解的类,才会进入该invoke方法.   

```
public Object invoke(final MethodInvocation methodInvocation) throws Throwable {
	// 获得目标类
	Class<?> targetClass =
		methodInvocation.getThis() != null ? AopUtils.getTargetClass(methodInvocation.getThis()) : null;
	
	// 获得方法	
	Method specificMethod = ClassUtils.getMostSpecificMethod(methodInvocation.getMethod(), targetClass);
	
	//  方法的签名不是Object
	if (specificMethod != null && !specificMethod.getDeclaringClass().equals(Object.class)) {
		// 获得方法信息
		final Method method = BridgeMethodResolver.findBridgedMethod(specificMethod);
		
		// 获得方法上是否有: @GlobalTransactional 注解
		final GlobalTransactional globalTransactionalAnnotation =
			getAnnotation(method, targetClass, GlobalTransactional.class);
			
		// 获得方法上是否有: @GlobalLock 注解
		final GlobalLock globalLockAnnotation = getAnnotation(method, targetClass, GlobalLock.class);
		
		// false
		boolean localDisable = disable || (degradeCheck && degradeNum >= degradeCheckAllowTimes);
		if (!localDisable) { // !false
			if (globalTransactionalAnnotation != null) { // @GlobalTransactional注解处理逻辑
				//  实际就是methodInvocation.proceed()方法的前后做拦截处理
				return handleGlobalTransaction(methodInvocation, globalTransactionalAnnotation);
			} else if (globalLockAnnotation != null) {  // @GlobalLock注解处理逻辑
				//  实际就是methodInvocation.proceed()方法的前后做拦截处理
				return handleGlobalLock(methodInvocation, globalLockAnnotation);
			}
		}
	}
	// 直接放过
	return methodInvocation.proceed();
}
```
### (4). GlobalTransactionalInterceptor.handleGlobalTransaction
```
// domain
// 事务模板,是不是感觉和Spring很像?
private final TransactionalTemplate transactionalTemplate = new TransactionalTemplate();
private final GlobalLockTemplate globalLockTemplate = new GlobalLockTemplate();


// method
Object handleGlobalTransaction(final MethodInvocation methodInvocation,
	final GlobalTransactional globalTrxAnno) throws Throwable {
	boolean succeed = true;
	try {
		// *********************************************************
		// 调用:TransactionalTemplate.execute方法,执行事务
		// TransactionalExecutor transactionalExecutor = new TransactionalExecutor(){...}
		// TransactionalTemplate transactionalTemplate = new TransactionalTemplate();
		// transactionalTemplate.execute(transactionalExecutor);
		// *********************************************************
		return transactionalTemplate.execute(new TransactionalExecutor() {
			
			// **************************************************
			// 
			// **************************************************
			@Override
			public Object execute() throws Throwable {
				return methodInvocation.proceed();
			}

			// **************************************************
			// 获得:注解@GlobalTransactional(name="xxx")
			// 如果没有配置,就用方法签名
			// **************************************************
			public String name() {
				String name = globalTrxAnno.name();
				if (!StringUtils.isNullOrEmpty(name)) {
					return name;
				}
				return formatMethod(methodInvocation.getMethod());
			}

			// **************************************************
			// 把注解@GlobalTransactional解析成业务模型对象:TransactionInfo
			// **************************************************
			@Override
			public TransactionInfo getTransactionInfo() {
				// reset the value of timeout
				int timeout = globalTrxAnno.timeoutMills();
				if (timeout <= 0 || timeout == DEFAULT_GLOBAL_TRANSACTION_TIMEOUT) {
					timeout = defaultGlobalTransactionTimeout;
				}

				TransactionInfo transactionInfo = new TransactionInfo();
				transactionInfo.setTimeOut(timeout);
				transactionInfo.setName(name());
				transactionInfo.setPropagation(globalTrxAnno.propagation());
				transactionInfo.setLockRetryInternal(globalTrxAnno.lockRetryInternal());
				transactionInfo.setLockRetryTimes(globalTrxAnno.lockRetryTimes());
				Set<RollbackRule> rollbackRules = new LinkedHashSet<>();
				for (Class<?> rbRule : globalTrxAnno.rollbackFor()) {
					rollbackRules.add(new RollbackRule(rbRule));
				}
				for (String rbRule : globalTrxAnno.rollbackForClassName()) {
					rollbackRules.add(new RollbackRule(rbRule));
				}
				for (Class<?> rbRule : globalTrxAnno.noRollbackFor()) {
					rollbackRules.add(new NoRollbackRule(rbRule));
				}
				for (String rbRule : globalTrxAnno.noRollbackForClassName()) {
					rollbackRules.add(new NoRollbackRule(rbRule));
				}
				transactionInfo.setRollbackRules(rollbackRules);
				return transactionInfo;
			}
		});
	} catch (TransactionalExecutor.ExecutionException e) {
		TransactionalExecutor.Code code = e.getCode();
		switch (code) {
			case RollbackDone:
				throw e.getOriginalException();
			case BeginFailure:
				succeed = false;
				failureHandler.onBeginFailure(e.getTransaction(), e.getCause());
				throw e.getCause();
			case CommitFailure:
				succeed = false;
				failureHandler.onCommitFailure(e.getTransaction(), e.getCause());
				throw e.getCause();
			case RollbackFailure:
				failureHandler.onRollbackFailure(e.getTransaction(), e.getOriginalException());
				throw e.getOriginalException();
			case RollbackRetrying:
				failureHandler.onRollbackRetrying(e.getTransaction(), e.getOriginalException());
				throw e.getOriginalException();
			default:
				throw new ShouldNeverHappenException(String.format("Unknown TransactionalExecutor.Code: %s", code));
		}
	} finally {
		if (degradeCheck) {
			EVENT_BUS.post(new DegradeCheckEvent(succeed));
		}
	}
}
```
### (5). 总结
> GlobalTransactionalInterceptor的职责如下:    
> 1. 拦截方法上的注解(@GlobalTransactional),委托给:TransactionalTemplate处理.  
> 2. 拦堆方法上的注解(@GlobalLock),委托给:GlobalLockTemplate处理.  
> 3. TransactionalTemplate和GlobalLockTemplate的业务逻辑,后面会再详解.   