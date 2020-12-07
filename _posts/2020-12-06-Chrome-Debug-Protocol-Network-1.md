---
layout: post
title: 'Chrome Debug Protocol--Network配置事件(五)'
date: 2020-12-06
author: 李新
tags: CDP
---

### (1). LogRequestsExample
```
final ChromeLauncher launcher = new ChromeLauncher();

// Launch chrome either as headless (true) or regular (false).
final ChromeService chromeService = launcher.launch(false);

// Create empty tab ie about:blank.
final ChromeTab tab = chromeService.createTab();

// Get DevTools service to this tab
final ChromeDevToolsService devToolsService = chromeService.createDevToolsService(tab);

// Get individual commands
final Page page = devToolsService.getPage();
// ***************************************************************
// 2. 从ChromeDevToolsService中获取:Network/Page
// ***************************************************************
final Network network = devToolsService.getNetwork();

// ***************************************************************
// 3. Network.onLoadingFinished
// ***************************************************************
// 为NetWork配置Event
network.onLoadingFinished(
    event -> {
      // Close the tab and close the browser when loading finishes.
      chromeService.closeTab(tab);
      launcher.close();
    });
// ... ...
```
### (2). ChromeDevToolsService.getNetwork
```
// Create invocation handler
CommandInvocationHandler commandInvocationHandler = new CommandInvocationHandler();

// Setup command cache for this session
Map<Method, Object> commandsCache = new ConcurrentHashMap<>();

// Create dev tools service.
// 根据:ChromeDevToolsServiceImpl产生动态的代理对象.它的所有抽象方法都会被代理.
ChromeDevToolsServiceImpl chromeDevToolsService = 
        ProxyUtils.createProxyFromAbstract(
        ChromeDevToolsServiceImpl.class,
        new Class[] {WebSocketService.class, ChromeDevToolsServiceConfiguration.class},
        new Object[] {webSocketService, chromeDevToolsServiceConfiguration},
        // ***********************************************************************
        // 设置:InvocationHandler,根据方法的返回值类型,再次创建动态代理对象.
        // ***********************************************************************
        (unused, method, args) ->
            commandsCache.computeIfAbsent(
                method,
                key -> {
                  // 获得方法的返回类型
                  // 根据返回类型,创建相应的代理对象
                  // 相应的返回类型全在这个package下:com.github.kklisura.cdt.protocol.commands
                  // Network/Page/DOM...
                  Class<?> returnType = method.getReturnType();
                  return ProxyUtils.createProxy(returnType, commandInvocationHandler);
        }));

// CommandInvocationHandler内剖持有:ChromeDevToolsServiceImpl
// ChromeDevToolsServiceImpl内部持有:WebSocketService
commandInvocationHandler.setChromeDevToolsService(chromeDevToolsService);
```
> 1. 对ChromeDevToolsService的抽象方法产生动态代理.动态代理的:InvocationHandler内容为:    
> 2. 当调用ChromeDevToolsService类的抽象方法时,根据抽象方法名称的返回类型,产生动态代理对象.   
> 3. 也就是说:NetWork也是一个动态代理对象,对应的:InvocationHandler为:CommandInvocationHandler

### (3). Network.onLoadingFinished
> 上面分析到了:Network是一个动态代理对象,所以,所有的业务操作都会在:CommandInvocationHandler.所以,可以预计:CommandInvocationHandler方法的一些设置,应该是对:ChromeDevToolsServiceImpl进行一些配置.  

```
network.onLoadingFinished();
```

> CommandInvocationHandler

```
// 定义事件的前缀
private static final String EVENT_LISTENER_PREFIX = "on";


public Object invoke(Object unused, Method method, Object[] args) throws Throwable {
  // method = EventListener onLoadingFinished(EventHandler)
  // 判断调用的方法是否为Event
  if (isEventSubscription(method)) {   // true
    // domainName = "Network"
    String domainName = method.getDeclaringClass().getSimpleName();
    // eventName = "loadingFinished"
    String eventName = getEventName(method);

    // eventHandlerType = LoadingFinished
    Class<?> eventHandlerType = getEventHandlerType(method);
    // *****************************************************************
    // 4. 委托给:ChromeDevToolsService.addEventListener
    // *****************************************************************
    return chromeDevToolsService.addEventListener(
        domainName, eventName, (EventHandler) args[0], eventHandlerType);
  }// end if  
}// end invoke

public static boolean isEventSubscription(Method method) {
    // name = "onLoadingFinished"
    String name = method.getName();
    // parameters = [com.github.kklisura.cdt.protocol.support.types.EventHandler<com.github.kklisura.cdt.protocol.events.network.LoadingFinished> arg0]
    Parameter[] parameters = method.getParameters();
    // 判断方法名称是否以:"on"开始,并且返回类型是否为:EventListener,同时,参数是否为:EventHandler
    return name.startsWith(EVENT_LISTENER_PREFIX)
        && EventListener.class.equals(method.getReturnType())
        && (parameters != null
            && parameters.length == 1
            && EventHandler.class.isAssignableFrom(parameters[0].getType()));
} // end isEventSubscription
```
### (4). ChromeDevToolsServiceImpl.addEventListener
```
// EventHolder
private Map<String, Set<EventListenerImpl>> eventNameToHandlersMap = new HashMap<>();


public EventListener addEventListener(
            // Network
            String domainName, 
            // loadingFinished
            String eventName, 
            // eventHandler为:network.onLoadingFinished()设置的lambad表达式
            // com.github.kklisura.cdt.examples.LogRequestsExample$$Lambda$18/2043106095
            EventHandler eventHandler, 
            // LoadingFinished
            Class<?> eventType) {
  // Network.loadingFinished
  String name = domainName + "." + eventName;
  // 创建:EventListenerImpl(注意:内部持有:ChromeDevToolsService/eventHandler)
  EventListenerImpl eventListener = new EventListenerImpl(name, eventHandler, eventType, this);

  // 如果名称(etwork.loadingFinished)对应在Set集合不存在,则先创建集合,再把EventListenerImpl添加到集合中,整个过程是元子性的.
  eventNameToHandlersMap.computeIfAbsent(name, this::createEventHandlerSet)
                        .add(eventListener);
  // 把创建的:EventListenerImpl返回
  return eventListener;
} // end addEventListener


private Set<EventListenerImpl> createEventHandlerSet(String unused) {
    return Collections.synchronizedSet(new HashSet<>());
}
```
### (5). 总结
> 1. 为NetWork配置监听事件(Network.onLoadingFinished(...)).      
> 2. 委托给:CommandInvocationHandler.invoke方法进行.      
> 3. CommandInvocationHandler根据方法名称是否为:on开始,委派给:ChromeDevToolsService.addEventListener.     
> ChromeDevToolsService.addEventListener方法对EventHandler进行包装成:EventListenerImpl,并添加到缓存中.     
> 5. <font color='red'>留个疑问:好像添加事件的过程,并没有与WebSocket有通信,需要继续往下跟踪.</font> 