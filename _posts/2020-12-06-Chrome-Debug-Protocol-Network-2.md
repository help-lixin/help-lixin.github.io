---
layout: post
title: 'Chrome Debug Protocol--Network enable(五)'
date: 2020-12-06
author: 李新
tags: CDP
---

### (1). LogRequestsExample
```
final ChromeLauncher launcher = new ChromeLauncher();

final ChromeService chromeService = launcher.launch(false);

final ChromeTab tab = chromeService.createTab();

final ChromeDevToolsService devToolsService = chromeService.createDevToolsService(tab);

final Page page = devToolsService.getPage();

final Network network = devToolsService.getNetwork();

network.onLoadingFinished(
    event -> {
      // Close the tab and close the browser when loading finishes.
      chromeService.closeTab(tab);
      launcher.close();
    });

// ***************************************************************
// 2. 启动NetWork
// ***************************************************************
network.enable();

// ... ...
```
### (2). Network.enable 
> 前面说过:NetWork为代理对象,所以:Network.enable --> CommandInvocationHandler.invoke

```

private static final AtomicLong ID_SUPPLIER = new AtomicLong(1L);


public Object invoke(Object unused, Method method, Object[] args) throws Throwable {
  // 针对配置事件
  if (isEventSubscription(method)) {
    // ...
  }

  // 
  // returnType = void
  Class<?> returnType = method.getReturnType();
  // 获取方法上的注解(@ReturnTypeParameter),是否有返回类型
  Class<?>[] returnTypeClasses = null;
  ReturnTypeParameter returnTypeParameter = method.getAnnotation(ReturnTypeParameter.class);
  if (returnTypeParameter != null) { // false
    returnTypeClasses = returnTypeParameter.value();
  }

  //获取方法上的注解(@Returns),是否有返回的属性
  String returnProperty = null;
  Returns returnsAnnotation = method.getAnnotation(Returns.class);
  if (returnsAnnotation != null) {// false
    returnProperty = returnsAnnotation.value();
  }

  // 创建方法的Invocation
  MethodInvocation methodInvocation = createMethodInvocation(method, args);
  // *****************************************************************
  // 3.最终委托给了:ChromeDevToolsService.invoke方法.
  // *****************************************************************
  return chromeDevToolsService.invoke(
      // null
      returnProperty, 
      // null
      returnType, 
      // void
      returnTypeClasses, 
      // MethodInvocation("Network.enable")
      methodInvocation);
}// end invoke


private MethodInvocation createMethodInvocation(
    Method method, 
    Object[] args) {

  // Network
  String domainName = method.getDeclaringClass().getSimpleName();
  // enable
  String methodName = method.getName();


  MethodInvocation methodInvocation = new MethodInvocation();
  // ID从1开始,进行自增
  methodInvocation.setId(ID_SUPPLIER.getAndIncrement());
  // Network.enable
  methodInvocation.setMethod(domainName + "." + methodName);
  // 构建入参:{}
  methodInvocation.setParams(buildMethodParams(method, args));
  return methodInvocation;
} //end createMethodInvocation

```
### (3). ChromeDevToolsService.invoke
```
public <T> T invoke(
      // null
      String returnProperty,
      // null
      Class<T> clazz,
      // null
      Class<?>[] returnTypeClasses,
      // MethodInvocation("Network.enable")
      MethodInvocation methodInvocation) {
    try {

      //  ***************InvocationResult数据结构如下:***************
      //  private String returnProperty;
      //  private JsonNode result;
      //  private boolean isSuccess = false;
      //  private CountDownLatch countDownLatch = new CountDownLatch(1);
      // ***********************************************************
      InvocationResult invocationResult = new InvocationResult(returnProperty);

      // ***********************************************************
      // 为什么要放到Map里?
      // 因为:发送报文给websocket后,
      // websocket返回数据,会包含着请求的ID,此时,根据id取value(InvocationResult),并value进行赋值或操作.   
      // key:为请求的唯一标识,value为:InvocationResult  
      // ***********************************************************
      invocationResultMap.put(methodInvocation.getId(), invocationResult);

      // 真正与websocket进行交互
      // {"id":1,"method":"Network.enable","params":{}}
      webSocketService.send(OBJECT_MAPPER.writeValueAsString(methodInvocation));

      // 判断是否有接受到返回内容
      // configuration.getReadTimeout() = 0
      // waitForResult当为0时,直接返回了:true
      boolean hasReceivedResponse =
          invocationResult.waitForResult(configuration.getReadTimeout(), TimeUnit.SECONDS);

      //  请求完成之后,删除key(唯一ID)
      invocationResultMap.remove(methodInvocation.getId());

      // ********************************************************
      // 那么,请求发送给weksocket,websocket又是如何处理的呢?
      // 这一部份内容,另开一章剖析
      // ********************************************************
      // 如果请求失败,则抛出异常
      if (!hasReceivedResponse) { //true
        throw new ChromeDevToolsInvocationException(
            "Timeout expired while waiting for server response.");
      }

      if (invocationResult.isSuccess()) { // 判断返回是否成功
        if (Void.TYPE.equals(clazz)) { // 如果返回类型为:void,直接return null
          return null;
        }

        if (returnTypeClasses != null) {
          return readJsonObject(returnTypeClasses, clazz, invocationResult.getResult());
        } else {
          return readJsonObject(clazz, invocationResult.getResult());
        }
      } else {   // 返回失败处理
        ErrorObject error = readJsonObject(ErrorObject.class, invocationResult.getResult());
        StringBuilder errorMessageBuilder = new StringBuilder(error.getMessage());
        if (error.getData() != null) {
          errorMessageBuilder.append(": ");
          errorMessageBuilder.append(error.getData());
        }

        throw new ChromeDevToolsInvocationException(
            error.getCode(), errorMessageBuilder.toString());
      }
    } catch (WebSocketServiceException e) {
      throw new ChromeDevToolsInvocationException("Failed sending web socket message.", e);
    } catch (InterruptedException e) {
      throw new ChromeDevToolsInvocationException("Interrupted while waiting response.", e);
    } catch (IOException ex) {
      throw new ChromeDevToolsInvocationException("Failed reading response message.", ex);
    }
} // end invoke
```
### (4). 总结
> 1. NetWork.enable方法的报文内容({"id":1,"method":"Network.enable","params":{}}).    
> 2. Network.enable方法会触发调用Chrome WebSocket.   
> 3. <font color='red'>既然存在:调用了websocket,那weksocket返回的结果过程是咋样的呢?需要另开一章来剖析.</font>     