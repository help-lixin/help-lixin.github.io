---
layout: post
title: 'Spring WebFlux源码(DispatcherHandler)'
date: 2020-12-18
author: 李新
tags: SpringWebFlux
---

### (1).DispatcherHandler初始化
> DelegatingWebFluxConfiguration负责webflux的配置.

```
@Configuration
public class DelegatingWebFluxConfiguration extends WebFluxConfigurationSupport {

} // end DelegatingWebFluxConfiguration


public class WebFluxConfigurationSupport implements ApplicationContextAware {
	// DispatcherHandler
	@Bean
	public DispatcherHandler webHandler() {
		return new DispatcherHandler();
	}
}
```
### (2). DispatcherHandler 类图
> 1. 实现了:WebHandler.   
> 2. 实现了:ApplicationContextAware,所以: DispatcherHandler被Spring初始化后,会回调相应的函数(setApplicationContext).  

!["DispatcherHandler类结构图"](/assets/spring-webflux/imgs/spring-webflux-dispatcher-handler-class.jpg)

### (3). DispatcherHandler
```
public class DispatcherHandler implements 
             WebHandler, 
			 // *****************************************
			 // Spring回调函数
			 // *****************************************
			 ApplicationContextAware {
   
   
   @Nullable
   private List<HandlerMapping> handlerMappings;

   @Nullable
   private List<HandlerAdapter> handlerAdapters;

   @Nullable
   private List<HandlerResultHandler> resultHandlers;


   @Override
	public void setApplicationContext(ApplicationContext applicationContext) {
		// ***********************************************
		// 这里不跟进了,无非不过就是从Spring容器中拿出Bean并赋值给当前对象的属性上.
		// ***********************************************
		initStrategies(applicationContext);
	}

	@Override
	public Mono<Void> handle(ServerWebExchange exchange) {
		if (this.handlerMappings == null) {
			return Mono.error(HANDLER_NOT_FOUND_EXCEPTION);
		}
		// handlerMappings = [RouterFunctionMapping, RequestMappingHandlerMapping, RoutePredicateHandlerMapping,SimpleUrlHandlerMapping]
		// 遍历:handlerMappings
		return Flux.fromIterable(this.handlerMappings)
		        // 调用:HandlerMapping.getHandler(exchage)返回一个:Mono<Object>
				.concatMap(mapping -> mapping.getHandler(exchange))
				// 判断是否存在
				.next()
				// 如果为空,抛出异常
				.switchIfEmpty(Mono.error(HANDLER_NOT_FOUND_EXCEPTION))
				// ***********************************************
				// 调用:mapping对应的handler方法
				// ***********************************************
				.flatMap(handler -> invokeHandler(exchange, handler))
				.flatMap(result -> handleResult(exchange, result));
	}// end handle

	private Mono<HandlerResult> invokeHandler(ServerWebExchange exchange, Object handler) {
		// handlerAdapters = [RequestMappingHandlerAdapter,HandlerFunctionAdapter,SimpleHandlerAdapter]
		if (this.handlerAdapters != null) {
			// ***********************************************
			// 判断handler是否支持
			// ***********************************************
			for (HandlerAdapter handlerAdapter : this.handlerAdapters) {
				// SimpleHandlerAdapter.supports(handler) == true
				if (handlerAdapter.supports(handler)) {
					return handlerAdapter.handle(exchange, handler);
				}
			}
		}
		return Mono.error(new IllegalStateException("No HandlerAdapter: " + handler));
	}// end invokeHandler

}
```
### (4).总结
> 1. DispatcherHandler接受所有的请求.   
> 2. 遍历内部(List)所有的:HandlerMapping,找到一个符合条件的:HandlerMapping. 
> 3. 遍历内部(List)所有的:HandlerAdapter,判断:HandlerMapping是否支持,如果受支持:则:HandlerAdapter.handler(ServerWebExchange,HandlerMapping). 

