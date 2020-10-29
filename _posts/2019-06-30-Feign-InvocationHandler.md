---
layout: post
title: 'Feign源码(InvocationHandler)'
date: 2019-06-30
author: 李新
tags: Feign
---

### (1).InvocationHandler
> 从前面的源码跟踪知道,Feign会解析接口上的注解,填充到MethodMetadata模型上,并根据target创建对应的代理对象

```
public class ReflectiveFeign extends Feign {
    public <T> T newInstance(Target<T> target) {
        // ...
        // InvocationHandlerFactory.Default
        // 1. 通过工厂创建代理对象的具体实现
        // methodToHandler为对应的handler
        InvocationHandler handler = factory.create(target, methodToHandler);
        T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(),new Class<?>[] {target.type()}, handler);
        // ...
    }
}
```
### (2).InvocationHandlerFactory.Default
```
public interface InvocationHandlerFactory {
    
    InvocationHandler create(Target target, Map<Method, MethodHandler> dispatch);
    
    interface MethodHandler {
        Object invoke(Object[] argv) throws Throwable;
    }
    
    static final class Default implements InvocationHandlerFactory {
        @Override
        public InvocationHandler create(Target target, Map<Method, MethodHandler> dispatch) {
          return new ReflectiveFeign.FeignInvocationHandler(target, dispatch);
        }
   } //end Default
}
```
### (3).ReflectiveFeign.FeignInvocationHandler
```
public class ReflectiveFeign extends Feign {
    // 2. 实现了JDK自带的java.lang.reflect.InvocationHandler
    static class FeignInvocationHandler 
           implements java.lang.reflect.InvocationHandler {
        private final Target target;
        private final Map<Method, MethodHandler> dispatch;
    
        FeignInvocationHandler(Target target, Map<Method, MethodHandler> dispatch) {
          this.target = checkNotNull(target, "target");
          this.dispatch = checkNotNull(dispatch, "dispatch for %s", target);
        }
    
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
          if ("equals".equals(method.getName())) {
            try {
              Object otherHandler =
                  args.length > 0 && args[0] != null ? Proxy.getInvocationHandler(args[0]) : null;
              return equals(otherHandler);
            } catch (IllegalArgumentException e) {
              return false;
            }
          } else if ("hashCode".equals(method.getName())) {
            return hashCode();
          } else if ("toString".equals(method.getName())) {
            return toString();
          }
          
          // ***********************************************************
          // 从dispatch中根据method,获得对应的:MethodHandler,并委托给它处理.
          // SynchronousMethodHandler methodHandler = dispatch.get(method);
          // Object obj = methodHandler.invoke(args);
          // return obj;
          // ***********************************************************
          return dispatch.get(method).invoke(args);
        } //end invoke
    }// end FeignInvocationHandler
}// end ReflectiveFeign
```
### (4).总结
> Feign根据targetType(HelloService)的类型,创建一个代理对象(FeignInvocationHandler(属于InvocationHandler的子类)).
> 由于在FeignInvocationHandler内部持有targetType所有method对应的handler.所以invoke只要根据method调用对应的handler即可.