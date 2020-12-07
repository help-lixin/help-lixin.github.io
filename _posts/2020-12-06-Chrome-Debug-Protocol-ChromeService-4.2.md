---
layout: post
title: 'Chrome Debug Protocol--ChromeService(2)(四)'
date: 2020-12-06
author: 李新
tags: CDP
---

### (1). LogRequestsExample
```
final ChromeLauncher launcher = new ChromeLauncher();

// Launch chrome either as headless (true) or regular (false).
final ChromeService chromeService = launcher.launch(false);

// 创建一个空的页面
// Create empty tab ie about:blank.
final ChromeTab tab = chromeService.createTab();

// **********************************************************************
// Get DevTools service to this tab
// 2. 通过:ChromeService,创建:ChromeDevToolsService
// **********************************************************************
final ChromeDevToolsService devToolsService = chromeService.createDevToolsService(tab);
```

### (2). ChromeTab数据结构

```
{
    "id":"F0B7F8A1A87A76E3E5BC0DCEB64A36B0",
    "parentId":"",
    "description":"",
    "title":"",
    "type":"page",
    "url":"about:blank",
    "devtoolsFrontendUrl":"/devtools/inspector.html?ws=localhost:54614/devtools/page/F0B7F8A1A87A76E3E5BC0DCEB64A36B0",
    "webSocketDebuggerUrl":"ws://localhost:54614/devtools/page/F0B7F8A1A87A76E3E5BC0DCEB64A36B0",
    "faviconUrl":""
}
```

### (3). ChromeService.createDevToolsService
> ChromeDevToolsServiceConfiguration内部持有读取超时,以及一个线程池对象.

```
public synchronized ChromeDevToolsService createDevToolsService(ChromeTab tab)
      throws ChromeServiceException {
    // **************************************************************************
    // 3. 委托给内部:ChromeService创建:ChromeDevToolsService
    // **************************************************************************
    return createDevToolsService(
        tab, 
        // 创建配置文件
        new ChromeDevToolsServiceConfiguration()
    );
}
```
### (4). ChromeService.createDevToolsService

```
public synchronized ChromeDevToolsService createDevToolsService(
      ChromeTab tab, ChromeDevToolsServiceConfiguration chromeDevToolsServiceConfiguration)
      throws ChromeServiceException {
    try {

      // 根据tabId进行缓存,如果tabId已经存在就从缓存中获取到:tab
      if (isChromeDevToolsServiceCached(tab)) { // false
        return getCachedChromeDevToolsService(tab);
      }

      // Connect to a tab via web socket
      // 从tab中获取debug url
      // ws://localhost:54614/devtools/page/F0B7F8A1A87A76E3E5BC0DCEB64A36B0
      String webSocketDebuggerUrl = tab.getWebSocketDebuggerUrl();

      // 调用工厂,创建连接
      // WebSocketServiceImpl.create(URI.create(webSocketDebuggerUrl))
      WebSocketService webSocketService =
          webSocketServiceFactory.createWebSocketService(webSocketDebuggerUrl);

      // ***************************************************************
      // 创建:InvocationHandler
      // ***************************************************************
      CommandInvocationHandler commandInvocationHandler = new CommandInvocationHandler();
      // Setup command cache for this session
      Map<Method, Object> commandsCache = new ConcurrentHashMap<>();


      // Create dev tools service.
      // **************************************************************
      // 4.委托给:javassist.util.proxy.ProxyFactory,创建动态代理(ChromeDevToolsServiceImpl).
      // **************************************************************
      // chromeDevToolsService = com.github.kklisura.cdt.services.impl.ChromeDevToolsServiceImpl_$$_jvst45f_0
      ChromeDevToolsServiceImpl chromeDevToolsService =
          ProxyUtils.createProxyFromAbstract(
              ChromeDevToolsServiceImpl.class,
              new Class[] {WebSocketService.class, ChromeDevToolsServiceConfiguration.class},
              new Object[] {webSocketService, chromeDevToolsServiceConfiguration},
              (unused, method, args) ->
                  // 根据调用的方法名称,判断在cache中是否存在
                  // 如果不存在,则调用函数(Function)处理.
                  commandsCache.computeIfAbsent(
                      method,
                      key -> {
                          // **************************************************
                          // 根据返回类型,创建对应的代理对象.代理对象的具体处理交给:
                          // CommandInvocationHandler,它属于:InvocationHandler的实现类.
                          // **************************************************
                        Class<?> returnType = method.getReturnType();
                        return ProxyUtils.createProxy(returnType, commandInvocationHandler);
                      })
            );

      // ****************************************************************************
      // 调用:ChromeDevToolsServiceImpl任何方法,会根据方法的返回类型,产生相应的动态代理对象
      // 动态对象的实现类为:InvocationHandler
      // ****************************************************************************
      
      // 给commandInvocationHandler配置:ChromeDevToolsServiceImpl
      // Register dev tools service with invocation handler.
      commandInvocationHandler.setChromeDevToolsService(chromeDevToolsService);

      // Cache it up.
      // 添加到缓存中.
      cacheChromeDevToolsService(tab, chromeDevToolsService);

      return chromeDevToolsService;
    } catch (WebSocketServiceException ex) {
      throw new ChromeServiceException("Failed connecting to tab web socket.", ex);
    }
} // end createDevToolsService
```
### (4). ProxyUtils.createProxyFromAbstract
```
public static <T> T createProxyFromAbstract(
      // 动态代理的类:
      // com.github.kklisura.cdt.services.impl.ChromeDevToolsServiceImpl
      Class<T> clazz, 
      
      // 动态代理类构造器需要的参数类型
      // [interface com.github.kklisura.cdt.services.WebSocketService, class com.github.kklisura.cdt.services.config.ChromeDevToolsServiceConfiguration]
      Class[] paramTypes, 
      
      // 动态代理类构造器需要的参数值
      // [com.github.kklisura.cdt.services.impl.WebSocketServiceImpl@332796d3, com.github.kklisura.cdt.services.config.ChromeDevToolsServiceConfiguration@4f0100a7]
      Object[] args, 

      // 相应的handler
      // com.github.kklisura.cdt.services.impl.ChromeServiceImpl$$Lambda$14/820537534@3cdf2c61
      InvocationHandler invocationHandler) {

    // 创建代理工厂 
    ProxyFactory proxyFactory = new ProxyFactory();
    // 设置基于哪个类创建代理工厂(com.github.kklisura.cdt.services.impl.ChromeDevToolsServiceImpl)
    proxyFactory.setSuperclass(clazz);
    // 只针对抽象方法进行代理.
    proxyFactory.setFilter(method -> Modifier.isAbstract(method.getModifiers()));
    try {
        return (T)
            proxyFactory.create(
                paramTypes,
                args,
                (o, method, method1, objects) -> invocationHandler.invoke(o, method, objects));
    } catch (Exception e) {
        LOGGER.error("Failed creating proxy from abstract class", e);
        throw new RuntimeException("Failed creating proxy from abstract class", e);
    }
} 
```
### (5). 查看ChromeDevToolsService类结构
!["查看ChromeDevToolsService类结构"](/assets/chrome/imgs/ChromeDevToolsServiceImpl-class.jpg)

### (6). 总结
> 1. 根据ChromeTab.getWebSocketDebuggerUrl()与WebSocket建立连接.   
> 2. 为ChromeDevToolsServiceImpl创建动态代理,在第5步,对类的结构剖析,我们知道ChromeDevToolsServiceImpl拥有大量的getXXX()方法.ChromeDevToolsServiceImpl内部持有:WebSocketService实例.        
> 3. 调用:ChromeDevToolsServiceImpl方法(Page page = devToolsService.getPage())时,<font color='red'>又会根据方法返回类型(Page),创建相应的动态代理对象(Page),调指定动态对象的处理类为:CommandInvocationHandler.</font>       
> 4. 为CommandInvocationHandler内部持有:ChromeDevToolsService实例.  
> 5. 所以,估计:后续所有的操作为(command)移交给:CommandInvocationHandler处理,而CommandInvocationHandler,又会把请求移交给:ChromeDevToolsService,而ChromeDevToolsService又会把请求移交给:WebSocketService.     