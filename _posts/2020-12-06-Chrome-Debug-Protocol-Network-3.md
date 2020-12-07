---
layout: post
title: 'Chrome Debug Protocol--Network enable后websocket如何接受数据(五)'
date: 2020-12-06
author: 李新
tags: CDP
---

### (1). 概述
> 前面的内容,剖析到了:Network.enable会向Chrome发送报文(websocket传输体),那么,websocket又是如何处理revice的呢?又是如何改变:InvocationResult对象的呢?   
> websocket接受数据的业务逻辑在:ChromeDevToolsServiceImpl的构造器里.   

### (2). Network.enable
```
// ***************************************************************
// 3. 启动NetWork,发送报文({"id":1,"method":"Network.enable","params":{}})
//    那websocket又是如何接受数据,并改变:InvocationResult对象的呢?
//    入口即在:ChromeDevToolsServiceImpl的构造器里
// ***************************************************************
network.enable();

// ... ...
```
### (3). ChromeDevToolsServiceImpl构造器
> 在调用:ChromeService.createDevToolsService方法时:
> 1. 创建了:WebSocketService,并与远程websocket建立起了连接.   
> 2. <font color='red'>通过动态代理,创建了ChromeDevToolsServiceImpl的实例,而ChromeDevToolsServiceImpl实例的构造器依赖:WebSocketService.</font>

```
public ChromeDevToolsServiceImpl(
      WebSocketService webSocketService, 
      ChromeDevToolsServiceConfiguration configuration)
  throws WebSocketServiceException {
  // websocket client  
  this.webSocketService = webSocketService;
  this.configuration = configuration;

  this.eventExecutorService = configuration.getEventExecutorService();

  this.closeLatch = new CountDownLatch(1);

  // ********************************************************************
  // 4. 为websocket配置Hnalder,而对应的业务为:ChromeDevToolsServiceImpl
  //    因为:ChromeDevToolsServiceImpl还实现了:Consumer<String>方法
  // ********************************************************************
  this.webSocketService.addMessageHandler(this);
}// end 构造器
```
### (4). WebSocketService.addMessageHandler

```
public void addMessageHandler(
    Consumer<String> consumer) throws WebSocketServiceException {
    // session在ChromeDevToolsService.createDevToolsService就已经初化了的.
    if (session == null) { // false
      throw new WebSocketServiceException(
          "You first must connect to ws server in order to receive messages.");
    }

    // 如果已经配置了MessageHandler,则抛出异常
    if (!session.getMessageHandlers().isEmpty()) { // false
      throw new WebSocketServiceException("You are already subscribed to this web socket service.");
    }

    // 配置MessageHandler
    session.addMessageHandler(
        new MessageHandler.Whole<String>() {
          @Override
          public void onMessage(String message) {
            LOGGER.debug("Received message {} on {}", message, session.getRequestURI());
            // *******************************************************
            // 5. 调用:consumer.accept == ChromeDevToolsServiceImpl.accept方法
            // *******************************************************
            consumer.accept(message);
          }
        });
}//end addMessageHandler
```
### (5). ChromeDevToolsServiceImpl.accept
```
public void accept(String message) {
  // 报文内容:
  // message = {"id":1,"result":{}}
    try {

      JsonNode jsonNode = OBJECT_MAPPER.readTree(message);
      // 获得ID
      JsonNode idNode = jsonNode.get(ID_PROPERTY);
      if (idNode != null) {
        // id = 1
        Long id = idNode.asLong();
        // 从Map中获取ID对应的:InvocationResult
        InvocationResult invocationResult = invocationResultMap.get(id);

        // 调用:InvocationResult相关方法,告之是否成功,以及报文内容.
        if (invocationResult != null) {
          JsonNode resultNode = jsonNode.get(RESULT_PROPERTY);
          JsonNode errorNode = jsonNode.get(ERROR_PROPERTY);

          if (errorNode != null) {
            invocationResult.signalResultReady(false, errorNode);
          } else {
            if (invocationResult.getReturnProperty() != null) {
              if (resultNode != null) {
                resultNode = resultNode.get(invocationResult.getReturnProperty());
              }
            }

            if (resultNode != null) {
              invocationResult.signalResultReady(true, resultNode);
            } else {
              invocationResult.signalResultReady(true, null);
            }
          }
        } else {
          LOGGER.warn(
              "Received result response with unknown invocation id {}. {}", id, jsonNode.asText());
        }
      } else {
        JsonNode methodNode = jsonNode.get(METHOD_PROPERTY);
        JsonNode paramsNode = jsonNode.get(PARAMS_PROPERTY);

        if (methodNode != null) {
          handleEvent(methodNode.asText(), paramsNode);
        }
      }
    } catch (IOException ex) {
      LOGGER.error("Failed reading web socket message!", ex);
    } catch (Exception ex) {
      LOGGER.error("Failed receiving web socket message!", ex);
    }
} // end accept
```
### (6).总结
> 1. ChromeDevToolsServiceImpl实现了:java.util.function.Consumer.   
> 2. ChromeDevToolsServiceImpl在创建时,会为:WebSocketService配置业务处理Handler,而这个Handler要求是:Consumer的子类,也就是:ChromeDevToolsServiceImpl它自己.    
> 3. ChromeDevToolsServiceImpl.accept方法,接受websocket返回数据后,对报文({"id":1,"result":{}})进行解码,根据ID,从Map中获得返回的数据结构(InvocationResult),设置是否成功,和报文体.    
