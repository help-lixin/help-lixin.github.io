---
layout: post
title: 'Reactor Stream接口分析'
date: 2020-10-26
author: 李新
tags: ReactorStream
---

### (1).Publisher
```
package org.reactivestreams;

public interface Publisher<T> {
    
    public void subscribe(Subscriber<? super T> s);
}
```
### (2).Subscriber
```
package org.reactivestreams;

public interface Subscriber<T> {
    public void onSubscribe(Subscription s);
    public void onNext(T t);
    public void onError(Throwable t);
    public void onComplete();    
}
```
### (3).Subscription
```
package org.reactivestreams;
public interface Subscription {
    public void request(long n);
    public void cancel();
}
```
### (4).总结
Publisher    :   发布者  
Subscriber   :   订阅者( 当调用Publisher.subscribe(Subscriber<? super T> s )时,会触发:Subscriber.onSubscribe(Subscription s) )  
Subscription  :   发布者与订阅者的一次订阅周期,一旦调用cancel去掉订阅,则发布者不会再推送消息.  