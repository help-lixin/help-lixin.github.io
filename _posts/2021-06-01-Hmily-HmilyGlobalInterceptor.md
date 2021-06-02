---
layout: post
title: 'Hmily 全局事务处理HmilyGlobalInterceptor(三)'
date: 2021-06-01
author: 李新
tags:  Hmily
---

### (1). 前言
> 前面对一些分布式事务框架(ServiceComb Pack和Seata)有一些剖析,其实,大家思路都差不多来着的,还是先找到注解@HmilyTCC(全局事务)的拦截器.

### (2). applicationContext.xml
```
<!-- 这个xml配置是在官网的案例中找到的 -->
<!-- 开启Aop --> 
<aop:aspectj-autoproxy expose-proxy="true"/>

<!-- 配置Aspect(对配置有注解@HmilyTCC和@HmilyTAC的类产生动代工理对象) -->
<bean id = "hmilyTransactionAspect" class="org.dromara.hmily.spring.aop.SpringHmilyTransactionAspect"/>
```
### (3). SpringHmilyTransactionAspect
```
package org.dromara.hmily.spring.aop;
public class SpringHmilyTransactionAspect 
       // ********************************************************
	   // 4. 继承
	   // ********************************************************
       extends AbstractHmilyTransactionAspect 
	   implements Ordered {
		   
    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE;
    }
	
} // end SpringHmilyTransactionAspect
```
### (4). AbstractHmilyTransactionAspect
```
package org.dromara.hmily.core.aspect;

// 1. 定义Aspect
@Aspect
public abstract class AbstractHmilyTransactionAspect {

	// 事务拦截器
    private final HmilyTransactionInterceptor interceptor = new HmilyGlobalInterceptor();

	// 2. 定义对@HmilyTCC和@HmilyTAC进行代码织入
    @Pointcut("@annotation(org.dromara.hmily.annotation.HmilyTCC) || @annotation(org.dromara.hmily.annotation.HmilyTAC)")
    public void hmilyInterceptor() {
    }

    
	// 3. 以环绕的形式
    @Around("hmilyInterceptor()")
    public Object interceptTccMethod(final ProceedingJoinPoint proceedingJoinPoint) throws Throwable {
		// 4. 委托给:HmilyTransactionInterceptor
        return interceptor.invoke(proceedingJoinPoint);
    }
	
} // end AbstractHmilyTransactionAspect
```
### (5). HmilyTransactionInterceptor
```
public class HmilyGlobalInterceptor implements HmilyTransactionInterceptor {
    
    private static RpcParameterLoader parameterLoader;
    
	// 1. TransTypeEnum可枚举的值有:TCC/TAC/XA/SXA/CC
    private static final EnumMap<TransTypeEnum, HmilyTransactionHandlerRegistry> REGISTRY = new EnumMap<>(TransTypeEnum.class);
    
	// 2. 通过:RpcParameterLoader产生HmilyTransactionContext
    static {
        parameterLoader = Optional.ofNullable(ExtensionLoaderFactory.load(RpcParameterLoader.class))
		                          .orElse(new LocalParameterLoader());
    }
    
	// 3. 注册tcc和tac
    static {
        REGISTRY.put(TransTypeEnum.TCC, ExtensionLoaderFactory.load(HmilyTransactionHandlerRegistry.class, "tcc"));
        REGISTRY.put(TransTypeEnum.TAC, ExtensionLoaderFactory.load(HmilyTransactionHandlerRegistry.class, "tac"));
    }
    
    @Override
    public Object invoke(final ProceedingJoinPoint pjp) throws Throwable {
		// 4. 从线程中加载上下文
        HmilyTransactionContext context = parameterLoader.load();
		// 调用invokeWithinTransaction
        return invokeWithinTransaction(context, pjp);
    }
    
    private Object invokeWithinTransaction(final HmilyTransactionContext context, final ProceedingJoinPoint point) throws Throwable {
		// 5. 拦截的目标方法
        MethodSignature signature = (MethodSignature) point.getSignature();
		
		// 5.1 根据方法的注解,获得:HmilyTransactionHandlerRegistry的实现.
        return getRegistry(signature.getMethod())
		       // ************************************************************
		       // 5.2 根据上下文,获得角色对应的HmilyTransactionHandler
			   //     每一个角色都对应一个:Handler
			   // ************************************************************
			   // START(1, "发起者")               -->  StarterHmilyTccTransactionHandler
			   // CONSUMER(2, "消费者")            --> ConsumeHmilyTccTransactionHandler
			   //  PARTICIPANT(3, "参与者")        --> ParticipantHmilyTccTransactionHandler
			   //  LOCAL(4, "本地调用")            --> LocalHmilyTccTransactionHandler
		       .select(context)
			   
			   // 5.3 调用HmilyTransactionHandler.handleTransaction进行处理.
			   // 6.我这里以:StarterHmilyTccTransactionHandler为例
			   .handleTransaction(point, context);
    }
    
    private HmilyTransactionHandlerRegistry getRegistry(final Method method) {
		// 根据方法上的注解,返回:HmilyTransactionHandlerRegistry的实现:
		// HmilyTccTransactionHandlerRegistry
		// HmilyTacTransactionHandlerRegistry
        return null != method.getAnnotation(HmilyTCC.class) 
		            ? REGISTRY.get(TransTypeEnum.TCC) 
					: REGISTRY.get(TransTypeEnum.TAC);
    }
```
### (6). StarterHmilyTccTransactionHandler
```
public class StarterHmilyTccTransactionHandler implements HmilyTransactionHandler, AutoCloseable {
	// 1. 事务执行器(单例模式)
	private final HmilyTccTransactionExecutor executor = HmilyTccTransactionExecutor.getInstance();
	    
	private DisruptorProviderManage<HmilyTransactionHandlerAlbum> disruptorProviderManage;
	
	public StarterHmilyTccTransactionHandler() {
		disruptorProviderManage = new DisruptorProviderManage<>(new HmilyTransactionExecutorHandler(),
				Runtime.getRuntime().availableProcessors() << 1, DisruptorProviderManage.DEFAULT_SIZE);
		disruptorProviderManage.startup();
	}
	
	public Object handleTransaction(final ProceedingJoinPoint point, final HmilyTransactionContext context)
	            throws Throwable {
		Object returnValue;
		Supplier<Boolean> histogramSupplier = null;
		
		// metrics
		Optional<MetricsHandlerFacade> handlerFacade = MetricsHandlerFacadeEngine.load();
		try {
			if (handlerFacade.isPresent()) {
				handlerFacade.get().counterIncrement(MetricsLabelEnum.TRANSACTION_TOTAL.getName(), TransTypeEnum.TCC.name());
				histogramSupplier = handlerFacade.get().histogramStartTimer(MetricsLabelEnum.TRANSACTION_LATENCY.getName(), TransTypeEnum.TCC.name());
			} 
			
			// *******************************************************************
			// 1. 准备执行
			// 1.1 创建全局事务(HmilyTransaction)
			// 1.2 获取@HmilyTCC注解上的信息,方法上的信息,入参类型,入参参数,对其,进行持久化(MySQL/MongoDB/Redis...).
			// 1.3 创建HmilyTransactionContext,绑定到Context
			// *******************************************************************
			HmilyTransaction hmilyTransaction = executor.preTry(point);
			try {
				
				// *******************************************************************
				// 2. 真正的执行目标方法
				// *******************************************************************
				//execute try
				returnValue = point.proceed();
				
				//  设置:HmilyTransactio的状成为:TRYING
				hmilyTransaction.setStatus(HmilyActionEnum.TRYING.getCode());
				
				// 3. 调用持久化(MySQL/MongoDB/Redis...)对象,更新状态
				executor.updateStartStatus(hmilyTransaction);
			} catch (Throwable throwable) {
				// *******************************************************************
				//if exception ,execute cancel
				// *******************************************************************
				final HmilyTransaction currentTransaction = HmilyTransactionHolder.getInstance().getCurrentTransaction();
				disruptorProviderManage.getProvider().onData(() -> {
					handlerFacade.ifPresent(metricsHandlerFacade -> metricsHandlerFacade.counterIncrement(MetricsLabelEnum.TRANSACTION_STATUS.getName(),
							TransTypeEnum.TCC.name(), HmilyRoleEnum.START.name(), HmilyActionEnum.CANCELING.name()));
					
					// *******************************************************************
					// 4. 异常时,回调:cancelMethod指定的方法,然后,发布事件:EventTypeEnum.REMOVE_HMILY_PARTICIPANT
					// @HmilyTCC(confirmMethod = "confirmOrderStatus", cancelMethod = "cancelOrderStatus")
					// *******************************************************************
					executor.globalCancel(currentTransaction);
					
				});
				throw throwable;
			}
			
			// *******************************************************************
			// 调用目标类的方法没有抛错的情况下,会调用:confirm
			// *******************************************************************
			//execute confirm
			final HmilyTransaction currentTransaction = HmilyTransactionHolder.getInstance().getCurrentTransaction();
			disruptorProviderManage.getProvider().onData(() -> {
				handlerFacade.ifPresent(metricsHandlerFacade -> metricsHandlerFacade.counterIncrement(MetricsLabelEnum.TRANSACTION_STATUS.getName(),
						TransTypeEnum.TCC.name(), HmilyRoleEnum.START.name(), HmilyActionEnum.CONFIRMING.name()));
				// *******************************************************************
				// 5. 正常时,回调:confirmOrderStatus指定的方法,然后,发布事件:EventTypeEnum.REMOVE_HMILY_PARTICIPANT
				// @HmilyTCC(confirmMethod = "confirmOrderStatus", cancelMethod = "cancelOrderStatus")
				// *******************************************************************
				executor.globalConfirm(currentTransaction);
			});
		} finally {
			HmilyContextHolder.remove();
			executor.remove();
			if (null != histogramSupplier) {
				histogramSupplier.get();
			}
		}
		return returnValue;
	} // end handleTransaction
	
	@Override
	public void close() {
		disruptorProviderManage.getProvider().shutdown();
	}
}
```
### (7). 总结
> 通过AOP,对目标方法(trye)进行代理,当成功的情况下,调用:confirmMethod,失败的情况下调用:cancelMethod   