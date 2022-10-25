---
layout: post
title: 'Zeebe ClusterServicesStep源码之CommandApiRequestHandler(十五)' 
date: 2022-10-14
author: 李新
tags:  Zeebe
---

### (1). 概述
在这一小篇主要剖析:CommandApiRequestHandler,它是RequestHandler的实现,主要用于接受请求,并做业务处理. 

### (2). GatewayBrokerTransportStep.startServerTransport
```
private void startServerTransport(
      final BrokerStartupContext brokerStartupContext,
      final ActorFuture<BrokerStartupContext> startupFuture) {

    final var concurrencyControl = brokerStartupContext.getConcurrencyControl();
    final var brokerInfo = brokerStartupContext.getBrokerInfo();
    final var schedulingService = brokerStartupContext.getActorSchedulingService();
    final var messagingService = brokerStartupContext.getApiMessagingService();

    // **********************************************************************
	// 1. 创建:AtomixServerTransport
	// **********************************************************************
    final var atomixServerTransport = new AtomixServerTransport(brokerInfo.getNodeId(), messagingService);
}
```

### (3). AtomixServerTransport.subscribe
> 配置topic的相应处理. 

```
// 此处RequestHandler为:CommandApiRequestHandler
public ActorFuture<Void> subscribe(final int partitionId, final RequestType requestType, final RequestHandler requestHandler) {
    return actor.call(
        () -> {
          final var topicName = topicName(partitionId, requestType);
          LOG.trace("Subscribe for topic {}", topicName);
          partitionsRequestMap.computeIfAbsent(partitionId, id -> new Long2ObjectHashMap<>());
		 
		 // ********************************************************************  
		 // 2. 调用NettyMessagingService.registerHandler配置处理器
		 // ********************************************************************  
          messagingService.registerHandler(
              topicName,
              (sender, request) ->
                  handleAtomixRequest(request, partitionId, requestType, requestHandler));
        }
		
		// 以下代码为上面相应代码的写法.
		// messagingService.registerHandler(
		//     topicName,
		//     new BiFunction<Address, byte[], CompletableFuture<byte[]>>(){
		//       public CompletableFuture<byte[]> apply( Address sender, byte[] request) {
		//	      // ... ...
		//       } 
		//    }
		// );
    ); // end return
}// end subscribe
```
### (4). NettyMessagingService.registerHandler
```
public void registerHandler(final String type, final BiFunction<Address, byte[], CompletableFuture<byte[]>> handler) {
	handlers.register(type,
	     // *******************************************************************
	     // 4. handler == CommandApiRequestHandler
	     //    handler.apply方法最终是调用上一步的:new BiFunction<Address, byte[], CompletableFuture<byte[]>>方法. 
		 //    在这里又包了一层,这代码相当于这样写.
		 //    new BiConsumer<ProtocolRequest, ServerConnection>(){
		 //	      public void accept(ProtocolRequest request, ServerConnection connection){
		 //          new BiFunction<Address, byte[], CompletableFuture<byte[]>>(){
		 //              public CompletableFuture<byte[]> apply( Address sender, byte[] request) {
		 //               	  handleAtomixRequest(... ...)
		 //              }
		 //          }
		 //       }
		 //   }
	     // *******************************************************************
		 (message, connection) -> 
           handler
		       .apply(message.sender(), message.payload())
		       .whenComplete(  // ... ... )
        // ... ...
    );		 
}	
```
### (6). AtomixServerTransport.handleAtomixRequest
```
private CompletableFuture<byte[]> handleAtomixRequest(
      final byte[] requestBytes,
      final int partitionId,
      final RequestType requestType,
      final RequestHandler requestHandler) {
    final var completableFuture = new CompletableFuture<byte[]>();
    actor.call(
        () -> {
          final var requestId = requestCount.getAndIncrement();
          final var requestMap = partitionsRequestMap.get(partitionId);
          if (requestMap == null) {
            final var errorMsg = String.format(ERROR_MSG_MISSING_PARTITON_MAP, partitionId);
            LOG.trace(errorMsg);
            completableFuture.completeExceptionally(new IllegalStateException(errorMsg));
            return;
          }

          try {
			//   requestHandler为:CommandApiRequestHandler
            requestHandler.onRequest(
                this,
                partitionId,
                requestId,
                new UnsafeBuffer(requestBytes),
                0,
                requestBytes.length);
            if (LOG.isTraceEnabled()) {
              LOG.trace(
                  "Handled request {} for topic {}",
                  requestId,
                  topicName(partitionId, requestType));
            }
            // we only add the request to the map after successful handling
            requestMap.put(requestId, completableFuture);
          } catch (final Exception exception) {
            LOG.error(
                "Unexpected exception on handling request for partition {}.",
                partitionId,
                exception);
            completableFuture.completeExceptionally(exception);
          }
        });

    return completableFuture;
}
```

### (5). CommandApiRequestHandler.handleExecuteCommandRequest
> CommandApiRequestHandler.onRequest方法,最终是委托给了:handleExecuteCommandRequest方法.  

```
private Either<ErrorResponseWriter, CommandApiResponseWriter> handleExecuteCommandRequest(
      final int partitionId,
      final long requestId,
      final CommandApiRequestReader reader,
      final CommandApiResponseWriter responseWriter,
      final ErrorResponseWriter errorWriter) {

    if (!isDiskSpaceAvailable) {
      return Either.left(errorWriter.outOfDiskSpace(partitionId));
    }

    final var command = reader.getMessageDecoder();
    final var logStreamWriter = leadingStreams.get(partitionId);
    final var limiter = partitionLimiters.get(partitionId);

    final var eventType = command.valueType();
    final var intent = Intent.fromProtocolValue(eventType, command.intent());
    final var event = reader.event();
    final var metadata = reader.metadata();

    metadata.requestId(requestId);
    metadata.requestStreamId(partitionId);
    metadata.recordType(RecordType.COMMAND);
    metadata.intent(intent);
    metadata.valueType(eventType);

    if (logStreamWriter == null) {
      errorWriter.partitionLeaderMismatch(partitionId);
      return Either.left(errorWriter);
    }

    if (event == null) {
      errorWriter.unsupportedMessage(
          eventType.name(), CommandApiRequestReader.RECORDS_BY_TYPE.keySet().toArray());
      return Either.left(errorWriter);
    }

    metrics.receivedRequest(partitionId);
    if (!limiter.tryAcquire(partitionId, requestId, intent)) {
      metrics.dropped(partitionId);
      LOG.trace(
          "Partition-{} receiving too many requests. Current limit {} inflight {}, dropping request {} from gateway",
          partitionId,
          limiter.getLimit(),
          limiter.getInflightCount(),
          requestId);
      errorWriter.resourceExhausted();
      return Either.left(errorWriter);
    }

    boolean written = false;
    try {
      // *************************************************************************
	  // 最终是写了日志流
	  // *************************************************************************
      written = writeCommand(command.key(), metadata, event, logStreamWriter);
      return Either.right(responseWriter);
    } catch (final Exception ex) {
      LOG.error("Unexpected error on writing {} command", intent, ex);
      errorWriter.internalError("Failed writing response: %s", ex);
      return Either.left(errorWriter);
    } finally {
      if (!written) {
        limiter.onIgnore(partitionId, requestId);
      }
    }
} // end 
```
### (6). RequestHandler
> 我们看下RequestHandler接口的签名,大概也就能理解它的职责了.

```
public interface RequestHandler {
	
	void onRequest(
	      ServerOutput serverOutput,
	      int partitionId,
	      long requestId,
	      DirectBuffer buffer,
	      int offset,
	      int length);
}	
```
### (7). 总结
Zeebe充份利用了Java8的lambed语法,相对来说是有一点点绕,当然,也有可能是我的能力有限,一时半会没理解他的代码.   