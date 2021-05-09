---
layout: post
title: 'Servicecomb Pack之全局事务@SagaStart(四)'
date: 2021-05-04
author: 李新
tags:  Servicecomb-Pack
---

### (1). 前言
> 在前面,把本地环境搭建起来了,在这里,开始对:Servicecomb Pack的源码进行剖析,在剖析之前,建议查看下各模块依赖图,能更加有利于你对模块的了解.    
> 很多的框架,在设计阶段,会对项目进行分层,层与层之间通过接口提供服务,然后,再通过门面或聚合所有的模块,形成服务.   
> 所以,<font color='red'>一个好的框架,它的分层模型决定了设计者对事物高层的次抽象(以及职能划分).</font> 

### (2). Servicecomb Pack 内部模块架构图
> 这张图虽然画的有点粗了点,但是,并不妨碍我们对Servicecomb Pack的了解.   
> 事务注解模块(Transaction Annotation),事务拦截器模块(Transaction Interceptor),事务上下文模块(Transaction Context),事务回调模块(Transaction Callback),事务执行器模块(Transaction Executor),事务传输模块(Transaction Transport)...

!["Servicecomb Pack 内部模块架构图"](/assets/servicecomb-pack/imgs/image-pack-system-archecture.png)

### (3). 如何入手?
> 1. ["先阅读参考入门手册"](https://github.com/help-lixin/servicecomb-pack/blob/master/docs/user_guide.md).  
> 2. 既然:Servicecomb Pack深度拥抱了Spring,自然,是要从它与Spring(EnableAutoConfiguration)的切入点跟踪.      
> 3. 所以,切入点模块为:omega-spring-starter.

```
# omega-spring-starter-0.6.0.jar/META-INF/spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=org.apache.servicecomb.pack.omega.spring.OmegaSpringAutoConfiguration
```
### (4). OmegaSpringAutoConfiguration
> OmegaSpringAutoConfiguration很简单,直接导入了两个@Configuration配置,重点关注:TransactionAspectConfig,因为一看名字就知道是AOP的聚合配置.  

```
@Configuration
// 很简单,就引用了两个@Configuration配置
// OmegaSpringConfig / TransactionAspectConfig
@Import({OmegaSpringConfig.class,TransactionAspectConfig.class})
@ConditionalOnProperty(value = {"omega.enabled"}, matchIfMissing = true)
public class OmegaSpringAutoConfiguration {
}
```
### (6). TransactionAspectConfig
> 通过翻看源码,定位到两个类:SagaStartAspect/TransactionAspect.  
> <font color='red'>SagaStartAspect: 针对配置有@SagaStart的类进行代理,这个注解是配置全局事务注解</font>  
> <font color='red'>TransactionAspect: 针对配置有@Compensable的类进行代理,这个注解是配置分支事务的注解</font>     

```
@Configuration
@EnableAspectJAutoProxy
public class TransactionAspectConfig {
	
	// 1. 对@Compensable注解内容进行检测
	@Bean
    CompensableAnnotationProcessor compensableAnnotationProcessor(
	      OmegaContext omegaContext,
	      @Qualifier("compensationContext") CallbackContext compensationContext) {
        return new CompensableAnnotationProcessor(omegaContext, compensationContext);
    }
	
	// 2. 对@Participate注解内容进行检测
	@Bean
    ParticipateAnnotationProcessor participateAnnotationProcessor(
	      OmegaContext omegaContext,
	      @Qualifier("coordinateContext") CallbackContext coordinateContext) {
	    return new ParticipateAnnotationProcessor(omegaContext, coordinateContext);
    } 
	
	// 3. CompensationMessageHandler消息处理器
	@Bean
    MessageHandler messageHandler(
	     SagaMessageSender sender,
		 @Qualifier("compensationContext") CallbackContext context, 
		 OmegaContext omegaContext) {
	    return new CompensationMessageHandler(sender, context);
	} 
	
	
	// ***************************************************************
	// 4. 针对:@SagaStart进行AOP拦截
	// ***************************************************************
	@Bean
    SagaStartAspect sagaStartAspect(SagaMessageSender sender, OmegaContext context) {
        return new SagaStartAspect(sender, context);
	}
	
	// ***************************************************************
	// 5. 针对@Compensable进行AOP拦截
	// ***************************************************************
	@Bean
	TransactionAspect transactionAspect(SagaMessageSender sender, OmegaContext context) {
       return new TransactionAspect(sender, context);
    }
	
	// 6. CoordinateMessageHandler
	@Bean
	TccMessageHandler coordinateMessageHandler(
	      TccMessageSender tccMessageSender,
	      @Qualifier("coordinateContext") CallbackContext coordinateContext,
	      OmegaContext omegaContext,
	      ParametersContext parametersContext) {
	    return new CoordinateMessageHandler(tccMessageSender, coordinateContext, omegaContext, parametersContext);
	}
	
	// ***************************************************************
	// 7. 针对@TccStart进行AOP拦截
	// ***************************************************************
	@Bean
	TccStartAspect tccStartAspect(
	      TccMessageSender tccMessageSender,
	      OmegaContext context) {
		return new TccStartAspect(tccMessageSender, context);
	}
	
	// ***************************************************************
	// 8. 针对@Participate进行拦截处理.
	// ***************************************************************
	@Bean
	TccParticipatorAspect tccParticipatorAspect(
	      TccMessageSender tccMessageSender,
	      OmegaContext context, ParametersContext parametersContext) {
	    return new TccParticipatorAspect(tccMessageSender, context, parametersContext);
	}
}
```
### (6). 全局事务处理(SagaStartAspect)
```
@Aspect
@Order(value = 100)
public class SagaStartAspect {
	private final SagaStartAnnotationProcessor sagaStartAnnotationProcessor;
	
	private final OmegaContext context;
	
	public SagaStartAspect(SagaMessageSender sender, OmegaContext context) {
		this.context = context;
	    this.sagaStartAnnotationProcessor = new SagaStartAnnotationProcessor(context, sender);
	}
	
	
	// 1. 拦截@SagaStart注解
	@Around("execution(@org.apache.servicecomb.pack.omega.context.annotations.SagaStart * *(..)) && @annotation(sagaStart)")
    Object advise(ProceedingJoinPoint joinPoint, SagaStart sagaStart) throws Throwable {
		// 创建全局事务
		initializeOmegaContext();
		// 判断是否开启了:akka,并且,配置的超时时间大于零
		if(context.getAlphaMetas().isAkkaEnabled() && sagaStart.timeout()>0){  // false
		  SagaStartAnnotationProcessorTimeoutWrapper wrapper = new SagaStartAnnotationProcessorTimeoutWrapper(this.sagaStartAnnotationProcessor);
		  return wrapper.apply(joinPoint,sagaStart,context);
		}else{ // true
		  // 2. 创建:SagaStartAnnotationProcessorWrapper
		  SagaStartAnnotationProcessorWrapper wrapper = new SagaStartAnnotationProcessorWrapper(this.sagaStartAnnotationProcessor);
		  // 3. apply
		  return wrapper.apply(joinPoint,sagaStart,context);
		}
		
	}
	
	// 创建全局事务ID
	private void initializeOmegaContext() {
	    context.setLocalTxId(context.newGlobalTxId());
	}
}
```
### (7). SagaStartAnnotationProcessorWrapper
```
public class SagaStartAnnotationProcessorWrapper {

  private static final Logger LOG = LoggerFactory.getLogger(MethodHandles.lookup().lookupClass());
  private final SagaStartAnnotationProcessor sagaStartAnnotationProcessor;

  public SagaStartAnnotationProcessorWrapper(
      SagaStartAnnotationProcessor sagaStartAnnotationProcessor) {
    this.sagaStartAnnotationProcessor = sagaStartAnnotationProcessor;
  }

  public Object apply(ProceedingJoinPoint joinPoint, SagaStart sagaStart, OmegaContext context)
      throws Throwable {
	// 获得目标类(BookingController)上的注解:@SagaStart	  
    Method method = ((MethodSignature) joinPoint.getSignature()).getMethod();
	// 1. 调用目标类的方法之前,先向Alpha发送SagaStartedEvent事件.
    sagaStartAnnotationProcessor.preIntercept(sagaStart.timeout());
	
    LOG.debug("Initialized context {} before execution of method {}", context, method.toString());
    try {
		// *******************************************************
		// 2. 调用目标类的方法
		// *******************************************************
      Object result = joinPoint.proceed();
	  // *******************************************************
	  // 3. 判断是否自动向:Alpha发送:SagaEndedEvent事件.
	  // *******************************************************
      if (sagaStart.autoClose()) {
        sagaStartAnnotationProcessor.postIntercept(context.globalTxId());
        LOG.debug("Transaction with context {} has finished.", context);
      } else {
        LOG.debug("Transaction with context {} is not finished in the SagaStarted annotated method.", context);
      }
      return result;
    } catch (Throwable throwable) {
	  // *******************************************************
	  // 4. 调用目标方法后,如果抛出的异常,不属于:OmegaException,则,向:Alpha发送:SagaAbortedEvent/TxAbortedEvent
	  // *******************************************************
      // We don't need to handle the OmegaException here
      if (!(throwable instanceof OmegaException)) {
        sagaStartAnnotationProcessor.onError(method.toString(), throwable);
        LOG.error("Transaction {} failed.", context.globalTxId());
      }
      throw throwable;
    } finally {
		// *******************************************************
		// 清除线程上绑定的全局事务信息.
		// *******************************************************
      context.clear();
    }
  }
}
```
### (8). @SagaStart全局事务流程图
!["@SagaStart全局事务流程图"](/assets/servicecomb-pack/imgs/SagaStart-Sequence-Diagram.jpg)

### (9). 总结
> 1. Spring对@SagaStart注解进行拦截(代表这个方法是一个全局事务的开始).  
> 2. 通过:OmegaContext创建一个全局事务ID.  
> 3. 调用目标类的方法之前,通过SagaStartAnnotationProcessor.preIntercept方法,向Alpha发送:SagaStartedEvent事件.   
> 4. 调用目标类和方法.   
> 5. 调用目标类的方法之后(没有异常的情况下),通过SagaStartAnnotationProcessor.postIntercept方法,向Alpha发送:SagaEndedEvent事件.   
> 6. 调用目标类的方法之后(有异常的情况下),通过SagaStartAnnotationProcessor.postIntercept方法,向Alpha发送:SagaAbortedEvent/TxAbortedEvent事件.  