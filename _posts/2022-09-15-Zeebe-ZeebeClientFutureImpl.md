---
layout: post
title: 'Zeebe源码之ZeebeClientFutureImpl(六)' 
date: 2022-09-15
author: 李新
tags:  Zeebe
---

### (1). 概述
在前面对ZeebeClient的构建过程,以及API有所了解了,这一小篇,对ZeebeClientFutureImpl的源码进行一个剖析,因为,该类,承载着Client与GRPC返回结果的中间件.  

### (2). ZeebeClientFutureImpl
```
public class ZeebeClientFutureImpl<ClientResponse, BrokerResponse>
     // ********************************************** 
	 // 1. 继承于:CompletableFuture,CompletableFuture是对Java多线程进行编排的工具.
	 //    该类在此处不进行详细介绍
	 // ********************************************** 
    extends CompletableFuture<ClientResponse>
	// ********************************************** 
	// 2. 实现了:ZeebeFuture
	// ********************************************** 
    implements ZeebeFuture<ClientResponse>,
	// ********************************************** 
	// 3. 实现了:io.grpc.stub.StreamObserver
	// ********************************************** 
	StreamObserver<BrokerResponse> {
    // ... ...
}		
```
### (3). ZeebeFuture
```
public interface ZeebeFuture<T> 
       //  *******************************************
	   // JDK基础,不进行深入的讲解了
	   // 继承于:java.util.concurrent.Future
	   //  *******************************************
       extends Future<T>, CompletionStage<T> {
	   // 为什么感觉这两个方法,象极了:CompletableFuture.get()/get(long timeout, TimeUnit unit)
	T join();
	
	T join(long timeout, TimeUnit unit);
}	
```
### (4). StreamObserver
```
// ************************************************
// GRPC的底层,也不进行深入了,但是这个接口的签名,代表了GRPC应该是一个Reactor(响应式流)的实现.
// ************************************************
public interface StreamObserver<V>  {
	
	void onNext(V value);
	
	void onError(Throwable t);
	
	void onCompleted();
}	
```
### (5). ZeebeClientFutureImpl
```
public class ZeebeClientFutureImpl<ClientResponse, BrokerResponse>
    extends CompletableFuture<ClientResponse>
    implements ZeebeFuture<ClientResponse>, StreamObserver<BrokerResponse> {

   private final Function<BrokerResponse, ClientResponse> responseMapper;
   
     public ZeebeClientFutureImpl() {
       this(brokerResponse -> null);
     }
   
     public ZeebeClientFutureImpl(final Function<BrokerResponse, ClientResponse> responseMapper) {
       this.responseMapper = responseMapper;
     }
   
     @Override
     public ClientResponse join() {
       try {
		 // *************************************************
		 // 实际是调用:CompletableFuture.get,在这里的结果实际来于自GRPC StreamObserver.onNext
		 // *************************************************
         return get();
       } catch (final ExecutionException e) {
        // ... ...
     }
   
     @Override
     public ClientResponse join(final long timeout, final TimeUnit unit) {
       try {
         // *************************************************
		 // 实际是调用:CompletableFuture.get(final long timeout, final TimeUnit unit)
		 // 在这里的结果实际来于自GRPC StreamObserver.onNext
		 // *************************************************
         return get(timeout, unit);
       } catch (final ExecutionException e) {
        // ... ...
       }
     }
   
     // *****************************************************************
	 // 重点:
	 // 在这里实际调用了:CompletableFuture.complete(Object)
	 // 为CompletableFuture配置了返回的结果,让我们可以通过CompletableFuture.get()方法获得返回的结果
	 // Zeebe真的是坑人不浅,用了CompletableFuture的API,却仅仅只是做了一个手工的设置结果. 
	 // *****************************************************************
     @Override
     public void onNext(final BrokerResponse brokerResponse) {
       try {
         complete(responseMapper.apply(brokerResponse));
       } catch (final Exception e) {
         completeExceptionally(e);
       }
     }
   
     @Override
     public void onError(final Throwable throwable) {
       completeExceptionally(throwable);
     }
   
     private RuntimeException transformExecutionException(final ExecutionException e) {
       final Throwable cause = e.getCause();
   
       if (cause instanceof StatusRuntimeException) {
         final Status status = ((StatusRuntimeException) cause).getStatus();
         throw new ClientStatusException(status, e);
       } else {
         throw new ClientException(e);
       }
     } // end transformExecutionException

}
```
### (6). 总结
在看ZeebeClientFutureImpl的源码时,看了接口的能力,会以为它会利用CompletableFuture进行线程的编排,结果,他并没有那么做,差点把自己给绕进去了. 