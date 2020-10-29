---
layout: post
title: 'Feign源码(ReflectiveFeign)'
date: 2019-06-30
author: 李新
tags: Feign
---

### (1).ReflectiveFeign
```
public class ReflectiveFeign extends Feign {
    public <T> T newInstance(Target<T> target) {
        // ************************************************
        // 调用ParseHandlersByName.apply()的方法对target的方法进行解析
        // key : beanName#methodName(paramType,paramType...)
        // value: MethodHandler
        // ************************************************
        Map<String, MethodHandler> nameToHandler = targetToHandlersByName.apply(target);
        // key:Method  value:MethodHandler
        Map<Method, MethodHandler> methodToHandler = new LinkedHashMap<Method, MethodHandler>();
        // 默认的方法处理(equals/hashcode...)
       List<DefaultMethodHandler> defaultMethodHandlers = new LinkedList<DefaultMethodHandler>();
        
        // 遍历target上的所有方法
        for (Method method : target.type().getMethods()) {
          if (method.getDeclaringClass() == Object.class) {
            continue;
          } else if (Util.isDefault(method)) {
            DefaultMethodHandler handler = new DefaultMethodHandler(method);
            defaultMethodHandlers.add(handler);
            methodToHandler.put(method, handler);
          } else {
            methodToHandler.put(method, nameToHandler.get(Feign.configKey(target.type(), method)));
          }
        }// end for
        
        // ***************************************************************
        // 根据InvocationHandler里面包含着一个Map
        // InvocationHandlerFactory.create(...)
        // map key:Method
        // map value:MethodHandler
        // ***************************************************************
        InvocationHandler handler = factory.create(target, methodToHandler);
        T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(),new Class<?>[] {target.type()}, handler);
        
        // 对默认方法进行处理
        for (DefaultMethodHandler defaultMethodHandler : defaultMethodHandlers) {
          defaultMethodHandler.bindTo(proxy);
        }// end for
        
        // 返回:代理对象
        return proxy;
  } //end newInstance
 
}// end ReflectiveFeign
```
### (2).总结
> ReflectiveFeign的newInstance方法,会**委托**给ParseHandlersByName的apply方法对target对象进行解析.最后返回Map结构(Map<String,MethodHandler>).    
> 再根据target创建Proxy对象(Proxy对象必须是一个InvocationHandler),而InvocationHandler内部维持着:Map<Method,MethodHandler>.可根据请求的Method,调用MethodHandler的invoker方法.