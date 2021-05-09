---
layout: post
title: 'Servicecomb Pack之分支事务@Compensable(七)'
date: 2021-05-04
author: 李新
tags:  Servicecomb-Pack
---

### (1). 前言
> 前面对全局事务进行了剖析了,从这一小节开始,对分支事务进行剖析.   

### (2). 如何入手?
> 注解:@Compensable(compensationMethod = "cancel")是切入点,所以,需要找到@Compensable是如何生效的.  

### (3). TransactionAspectConfig
```
@Configuration
@EnableAspectJAutoProxy
public class TransactionAspectConfig {
	
	@Bean
	TransactionAspect transactionAspect(SagaMessageSender sender, OmegaContext context) {
		return new TransactionAspect(sender, context);
	}
}
```
### (4). TransactionAspect
```
public class TransactionAspect extends TransactionContextHelper {

  private static final Logger LOG = LoggerFactory.getLogger(MethodHandles.lookup().lookupClass());

  private final OmegaContext context;

  private final CompensableInterceptor interceptor;

  public TransactionAspect(SagaMessageSender sender, OmegaContext context) {
    this.context = context;
    this.context.verify();
	// ********************************************************************
	// 1. 在创建构造器时,创建了拦截器(CompensableInterceptor) 
	// ********************************************************************
    this.interceptor = new CompensableInterceptor(context, sender);
  }

  // 2. 定义切面,拦截:@Compensable注解的请求
  @Around("execution(@org.apache.servicecomb.pack.omega.transaction.annotations.Compensable * *(..)) && @annotation(compensable)")
  Object advise(ProceedingJoinPoint joinPoint, Compensable compensable) throws Throwable {
	// 3. 获得目标类的方法(我这里是:CarBookingService.order)  
    Method method = ((MethodSignature) joinPoint.getSignature()).getMethod();
	
	// 这种场景,主要是解决:上下文(OmegaContext)通过参数进行传递.
	// 4. 提取目标类方法上的参数,判断参数是否为:TransactionContext/TransactionContextProperties
	//  如果是:TransactionContextProperties,则创建:TransactionContext WrapperTransactionContextProperties.
    TransactionContext transactionContext = extractTransactionContext(joinPoint.getArgs());
	
	// 5. 提取参数,绑定到线程上下文里.
    if (transactionContext != null) {
      populateOmegaContext(context, transactionContext);
    }
    
	// 6. 如果全局事务ID为空,则抛出异常.
    if (context.globalTxId() == null) {
      throw new OmegaException("Cannot find the globalTxId from OmegaContext. Please using @SagaStart to start a global transaction.");
    }
	
	// 7. 提取本地事务ID,进行暂存(与后面:finally代码块相呼应)
    String localTxId = context.localTxId();
	
	// 8. 创建新的本地事务ID
    context.newLocalTxId();
    LOG.debug("Updated context {} for compensable method {} ", context, method.toString());

	// 9. 获取注解上的重试次数(我这里是默认值:0)
    int forwardRetries = compensable.forwardRetries();
	
	// 10. 根据重试策略,创建:RecoveryPolicy
	// /**
    //  * If retries == 0, use the default recovery to execute only once.
    //  * If retries > 0, it will use the forward recovery and retry the given times at most.
    // */
    RecoveryPolicy recoveryPolicy = RecoveryPolicyFactory.getRecoveryPolicy(forwardRetries);
    try {
	
		// 	***************************************************
		// 11. 委托给:RecoveryPolicy进行处理
		// 	***************************************************
        return recoveryPolicy.apply(joinPoint, compensable, interceptor, context, localTxId, forwardRetries);
		
	} finally {
	  // *****************************************************
	  // 9. 重新设置本地事务ID为,上次暂存的:localTxId
	  // *****************************************************
      context.setLocalTxId(localTxId);
      LOG.debug("Restored context back to {}", context);
    }
  }

  @Override
  protected Logger getLogger() {
    return LOG;
  }

}
```
### (5). RecoveryPolicy
```
// 1. RecoveryPolicy就定义了一个接口.
public interface RecoveryPolicy {
  Object apply(ProceedingJoinPoint joinPoint, Compensable compensable, CompensableInterceptor interceptor,
      OmegaContext context, String parentTxId, int forwardRetries) throws Throwable;
} // end RecoveryPolicy

// 2. AbstractRecoveryPolicy会根据注解(@Compensable),获取:forwardTimeout,走不同的策略
//   forwardTimeout > 0   委托给:RecoveryPolicyTimeoutWrapper处理
//   forwardTimeout == 0  委托给子类:DefaultRecovery处理
//   我这里以:DefaultRecovery为例
public abstract class AbstractRecoveryPolicy implements RecoveryPolicy {

  // ***************************************************************
  // 3. 再次定义抽象(applyTo)方法,给子类
  // ***************************************************************
  public abstract Object applyTo(ProceedingJoinPoint joinPoint, Compensable compensable,
      CompensableInterceptor interceptor, OmegaContext context, String parentTxId, int forwardRetries)
      throws Throwable;

  @Override
  public Object apply(ProceedingJoinPoint joinPoint, Compensable compensable,
      CompensableInterceptor interceptor, OmegaContext context, String parentTxId, int forwardRetries)
      throws Throwable {
    Object result;
    if(compensable.forwardTimeout()>0){
      RecoveryPolicyTimeoutWrapper wrapper = new RecoveryPolicyTimeoutWrapper(this);
      result = wrapper.applyTo(joinPoint, compensable, interceptor, context, parentTxId, forwardRetries);
    } else {
      result = this.applyTo(joinPoint, compensable, interceptor, context, parentTxId, forwardRetries);
    }
    return result;
  }// end apply
} // end AbstractRecoveryPolicy


// 4. DefaultRecovery
public class DefaultRecovery extends AbstractRecoveryPolicy {
  private static final Logger LOG = LoggerFactory.getLogger(MethodHandles.lookup().lookupClass());

  @Override
  public Object applyTo(ProceedingJoinPoint joinPoint, Compensable compensable, CompensableInterceptor interceptor,
      OmegaContext context, String parentTxId, int forwardRetries) throws Throwable {
    // 5. 获得目标类的方法
    Method method = ((MethodSignature) joinPoint.getSignature()).getMethod();
    LOG.debug("Intercepting compensable method {} with context {}", method.toString(), context);

	// 6. 获得目标类上的补偿方法信息
    String compensationSignature =
        compensable.compensationMethod().isEmpty() ? "" : compensationMethodSignature(joinPoint, compensable, method);

    String retrySignature = (forwardRetries != 0 || compensationSignature.isEmpty()) ? method.toString() : "";
	
	// ****************************************************************************************
	// 7. 向Alpha Server发送事件(TxStartedEvent)
	// ****************************************************************************************
    AlphaResponse response = interceptor.preIntercept(
	                             parentTxId, 
								 compensationSignature, 
								 compensable.forwardTimeout(),
								 retrySignature, 
								 forwardRetries, 
								 compensable.forwardTimeout(),
								 compensable.reverseRetries(), 
								 compensable.reverseTimeout(), 
								 compensable.retryDelayInMilliseconds(), 
								 // *****************************************************************
								 // 8. 在向
								 // *****************************************************************
								 joinPoint.getArgs());
    
	// 8. 向Alpha Server发送事件失败的情况下,都不需要往下走流程了
	if (response.aborted()) {
      String abortedLocalTxId = context.localTxId();
      context.setLocalTxId(parentTxId);
      throw new InvalidTransactionException("Abort sub transaction " + abortedLocalTxId +
          " because global transaction " + context.globalTxId() + " has already aborted.");
    } 

    try {
		
	   // *******************************************************************
	   // 9. 调用目标类的方法(CarBookingService.order)
	   // *******************************************************************
      Object result = joinPoint.proceed();
	  
	  // *******************************************************************
	  // 10. 调用目标类的方法,没有抛出异常的情况下:向Alpha Server发送事件(TxEndedEvent)
	  // *******************************************************************
      interceptor.postIntercept(parentTxId, compensationSignature);

      return result;
    } catch (Throwable throwable) {
	   // *******************************************************************
	   // 11. 调用目标类的方法,抛出异常的情况下:向Alpha Server发送事件(TxAbortedEvent)
       // *******************************************************************
      if (compensable.forwardRetries() == 0 || (compensable.forwardRetries() > 0
          && forwardRetries == 1)) {
        interceptor.onError(parentTxId, compensationSignature, throwable);
      }
      throw throwable;
    }
  }

  String compensationMethodSignature(ProceedingJoinPoint joinPoint, Compensable compensable, Method method)
      throws NoSuchMethodException {
    return joinPoint.getTarget()
        .getClass()
        .getDeclaredMethod(compensable.compensationMethod(), method.getParameterTypes())
        .toString();
  }
}
```
### (6). 总结
> 1. WebConfig配置的拦截器(TransactionHandlerInterceptor),会从协议头里获得全局事务和分支事务,绑定到当前线程里.   
> 2. 对@Compensable注解进行代理.   
> 3. <font color='red'>调用目标对象之前,先向Alpha Server发布事件(TxStartedEvent).</font>   
> 4. <font color='red'>调用目标对象的方法.</font>  
> 5. <font color='red'>调用目标对象之后(没有抛出异常的情况下),向Alpha Server发布事件(TxEndedEvent).</font>   
> 6. <font color='red'>调用目标对象之后(抛出异常的情况下),向Alpha Server发布事件(TxAbortedEvent),注意:仅仅是发布事件,调用补偿并不在这个逻辑里.</font>   
> 7. 正向流程分析完了,是否有疑问?补偿流程是什么时候做的?答案是:补偿操作是由Alpha Server回调Omega,所以,会有相应的Omega会有Handler来处理,后面会详细剖析这一部份.   