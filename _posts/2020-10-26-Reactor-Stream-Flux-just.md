---
layout: post
title: 'Reactor Stream源码(Flux.just)'
date: 2020-10-26
author: 李新
tags: ProjectReactorStream
---

### (1).Flux类关系图
```
// 注意该类是抽象类
public abstract class Flux<T> 
               // 实现了reactor.core.CorePublisher
               // CorePublisher是Publisher的增加
               implements CorePublisher<T> {
}

public interface CorePublisher<T> 
       // org.reactivestreams.Publisher
       extends Publisher<T> {
    
}

```

### (2).Flux.just("1","2")
```
public static <T> Flux<T> just(T... data) {
    // ["1","2"]
    return fromArray(data);
}
```
### (3).Flux.fromArray
```
public static <T> Flux<T> fromArray(T[] array) {
    // ["1","2"]
    if (array.length == 0) { // false
        return empty();
    }
    if (array.length == 1) { // false
        return just(array[0]);
    }
    
    // 创建FluxArray包裹数组
    // 调用onAssembly
    return onAssembly(new FluxArray<>(array));
}
```
### (4).FluxArray
> FluxArray既是Flux的子类,而且还实现了:Publisher

```
final class FluxArray<T> 
      // 继承于:reactor.core.publisher.Flux
      extends Flux<T> 
      // reactor.core.Fuseable
      implements Fuseable, 
      // reactor.core.publisher.SourceProducer
      SourceProducer<T> {
    
}

interface SourceProducer<O> 
     // reactor.core.Scannable
     extends Scannable, 
     // org.reactivestreams.Publisher
             Publisher<O> {
}
```
### (5). Flux.onAssembly
```
// onAssembly 包裹结果集,可以Hooks可以TRACE整个过程
protected static <T> Flux<T> onAssembly(Flux<T> source) {
    // source = FluxArray
    Function<Publisher, Publisher> hook = Hooks.onEachOperatorHook;
    if(hook != null) { // false
        source = (Flux<T>) hook.apply(source);
    }
    
    if (Hooks.GLOBAL_TRACE) {
            AssemblySnapshot stacktrace = new AssemblySnapshot(null, Traces.callSiteSupplierFactory.get());
            source = (Flux<T>) Hooks.addAssemblyInfo(source, stacktrace);
    }
    return source;
}
```
### (6).总结
1. Flux.just会把数组包裹成:reactor.core.publisher.FluxArray
2. 返回:reactor.core.publisher.FluxArray,因为FluxArray继承于:Flux并实现了:Publisher