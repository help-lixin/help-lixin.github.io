---
layout: post
title: 'Servicecomb Pack之分支事务@Compensable(八)'
date: 2021-05-04
author: 李新
tags:  Servicecomb-Pack
---

### (1). 前言
> 前面对分支事务的正向流程进行了剖析,这一小节,对补偿操作流程进行分析.  

### (2). 如何入手?
> 当调用目标类(业务)出现异常时,会调用:CompensableInterceptor.onError方法,向Alpha发布TxAbortedEvent事件.    
> 可是在这里,仅仅是发布了一件事件(TxAbortedEvent),那补偿操作是怎么做的呢?  
> 通过框架图,就能看出来:<font color='red'>补偿操作是由:Alpha收到事件后,调用Omega而触发的补偿.</font>     
> 怎么跟踪?<font color='red'>让分支事务抛出异常,在分支事务的补偿处打上断点,查看整个调用链即可(寻找:Netty Handler的入口).</font>  

!["CompensationMessageHandler为补偿操作入口"](/assets/servicecomb-pack/imgs/CompensationMessageHandler.jpg)
### (3). CompensationMessageHandler初始化

```
// 1. OmegaSpringConfig配置
@Configuration
class OmegaSpringConfig {
	
	// 2. 创建:CallbackContext
	@Bean(name = {"compensationContext"})
	CallbackContext compensationContext(OmegaContext omegaContext, SagaMessageSender sender) {
		return new CallbackContext(omegaContext, sender);
	} // end 
}

// 3. TransactionAspectConfig配置
@Configuration
@EnableAspectJAutoProxy
public class TransactionAspectConfig {
	@Bean
	MessageHandler messageHandler(
	          SagaMessageSender sender,
			  // *****************************************************************
			  // 4. 注入第2步的:CallbackContext
			  // *****************************************************************
			  @Qualifier("compensationContext") CallbackContext context, 
			  OmegaContext omegaContext) {
		return new CompensationMessageHandler(sender, context);
	}
}
```
### (4). CompensationMessageHandler
```
public class CompensationMessageHandler implements MessageHandler {
  private final SagaMessageSender sender;

  private final CallbackContext context;

  public CompensationMessageHandler(SagaMessageSender sender, CallbackContext context) {
    this.sender = sender;
    this.context = context;
  }

  @Override
  public void onReceive(String globalTxId, String localTxId, String parentTxId, String compensationMethod,
      Object... payloads) {
	// 接收消处,委派给:CallbackContext
    // 话说:从这个类的名字上来看,这个类就是一个Callback,为什么变量命名是context?
	// 会让人一眼就误解.
    context.apply(globalTxId, localTxId, parentTxId, compensationMethod, payloads);
	
    if (!context.getOmegaContext().getAlphaMetas().isAkkaEnabled()) { // true
	  // 向Alpha发送事件(TxCompensatedEvent)
      sender.send(new TxCompensatedEvent(globalTxId, localTxId, parentTxId, compensationMethod));
    }
  }
}
```
### (5). CallbackContext
```
public class CallbackContext {
  private static final Logger LOG = LoggerFactory.getLogger(MethodHandles.lookup().lookupClass());

  private final Map<String, CallbackContextInternal> contexts = new ConcurrentHashMap<>();
  private final OmegaContext omegaContext;
  private final SagaMessageSender sender;

  public CallbackContext(OmegaContext omegaContext, SagaMessageSender sender) {
    this.omegaContext = omegaContext;
    this.sender = sender;
  }
  
  // 1. Spring容器在,启动时:
  // 1.1 扫描到@Compensable注解
  // 1.2 解析@Compensable上的属性(compensationMethod)
  // 1.3 通过CallbackContext持有所有补偿对象(cancel).
  //    @Compensable(compensationMethod = "cancel")
  //    void order(CarBooking booking) {} 
  //    void cancel(CarBooking booking) {}
  public void addCallbackContext(String key, Method compensationMethod, Object target) {
    compensationMethod.setAccessible(true);
    contexts.put(key, new CallbackContextInternal(target, compensationMethod));
  }

  // 2. 回调处理
  public void apply(
                 // 全局事务ID
                 String globalTxId, 
				 // 分支事务ID
				 String localTxId, 
				 // 父事务ID(这里要留个疑问,Saga对于事务不是只能是嵌套两层吗?莫非,华为去掉了这个限制?)
				 String parentTxId, 
				 // 补偿的方法名称(cancel)
				 String callbackMethod, 
				 // 补偿时的参数
				 Object... payloads) {
    
	// 根据方法名称,获得目标对象和目标方法
	CallbackContextInternal contextInternal = contexts.get(callbackMethod);
	
	// 从线程上下文获得全局事务ID(要结合finally看,会重新还原,线程上下文的信息)
    String oldGlobalTxId = omegaContext.globalTxId();
	// 从线程上下文获得分支事务ID(要结合finally看,会重新还原,线程上下文的信息)
    String oldLocalTxId = omegaContext.localTxId();
    try {
	  // 重新为线程上下文设置:全局事务ID
      omegaContext.setGlobalTxId(globalTxId);
	  // 重新为线程上下文设置:分支事务ID
      omegaContext.setLocalTxId(localTxId);
	  
	  // ************************************************************
	  // 2.1 调用业务对象(目标类)的补偿方法(CarBookingService.cancel)
	  // ************************************************************
      contextInternal.callbackMethod.invoke(contextInternal.target, payloads);
	
	  // 判为是否启用:akka	
      if (omegaContext.getAlphaMetas().isAkkaEnabled()) { // false
        sender.send( new TxCompensateAckSucceedEvent(omegaContext.globalTxId(), omegaContext.localTxId(),
                parentTxId, callbackMethod));
      }
      LOG.info("Callback transaction with global tx id [{}], local tx id [{}]", globalTxId, localTxId);
    } catch (IllegalAccessException | InvocationTargetException e) {  // ****************只处理,这两类异常?*****************
	  // 这段逻辑,仅在AKKA时,才有效果,AKKA后面再研究,Akka是0.5版本引入的,难道在0.5版本之前,补偿重试功能一直是有Bug的吗?
	  if (omegaContext.getAlphaMetas().isAkkaEnabled()) { // false
        sender.send(
            new TxCompensateAckFailedEvent(omegaContext.globalTxId(), omegaContext.localTxId(),
                parentTxId, callbackMethod, e));
      }
	  
	  // *********************************************************
	  // 补偿方法抛出了异常,仅仅是打印日志.
	  // 这样子的话:
	  //  1. cacel抛出了异常,Alpha是否做重试?(经测试,我发现没有重试功能.)
	  //  2. 如果:Alpha不做重式,那么,开发人员在cancel方法内必须不能抛出异常,否则,会造成,分支事务不一致性.  
	  //  3. 感觉不合逻辑哈.为什么不重试?
	  // *********************************************************
      LOG.error(
          "Pre-checking for callback method " + contextInternal.callbackMethod.toString()
              + " was somehow skipped, did you forget to configure callback method checking on service startup?", e);
    } finally {
	  // 还原线程上下文信息.
      omegaContext.setGlobalTxId(oldGlobalTxId);
      omegaContext.setLocalTxId(oldLocalTxId);
    }
  }

  public OmegaContext getOmegaContext() {
    return omegaContext;
  }

  private static final class CallbackContextInternal {
    private final Object target;

    private final Method callbackMethod;

    private CallbackContextInternal(Object target, Method callbackMethod) {
      this.target = target;
      this.callbackMethod = callbackMethod;
    }
  }
}
```
### (6). 总结
> 通过上面的分析,能得到结论:    
> 1. 补偿方法是由Alpha触发Omega去执行的.  
> 2. <font color='red'>ServiceComb Pack还存在严重的Bug,不能用于生产环境.</font>      