---
layout: post
title: 'Reactor Stream源码(Flux.subscribe)'
date: 2020-10-26
author: 李新
tags: ProjectReactorStream
---

### (1).案例
```
Flux.just("1","2")
    .subscribe(c->{
            System.out.println("consumer:" + c);
    });
```
### (2).Flux.subscribe
```
public final Disposable subscribe(Consumer<? super T> consumer) {
    // consumer = help.lixin.samples.Flux2Test$$Lambda$7
    Objects.requireNonNull(consumer, "consumer");
    return subscribe(consumer, null, null);
}
```
### (3).Flux.subscribe
```
public final Disposable subscribe(
        @Nullable Consumer<? super T> consumer,
        @Nullable Consumer<? super Throwable> errorConsumer,
        @Nullable Runnable completeConsumer) {
           // consumer 订阅
           // errorConsumer 错误处理
           // completeConsumer 完成处理
    return subscribe(consumer, errorConsumer, completeConsumer, (Context) null);
}
```
### (4).Flux.subscribe
```
public final Disposable subscribe(
        @Nullable Consumer<? super T> consumer,
        @Nullable Consumer<? super Throwable> errorConsumer,
        @Nullable Runnable completeConsumer,
        // null
        @Nullable Context initialContext) {
    return subscribeWith(
        // 创建:reactor.core.publisher.LambdaSubscriber
        // 包课着正确消费者/错误消费者/完成处理等...
        new LambdaSubscriber<>(consumer, errorConsumer,completeConsumer,null,initialContext)
    );
}
```
### (5).LambdaSubscriber
```
interface CoreSubscriber<T> extends Subscriber<T> {}

interface InnerConsumer<I>
          extends CoreSubscriber<I>, 
          Scannable {             
}

final class LambdaSubscriber<T>
		implements InnerConsumer<T>, 
             Disposable {      
}
// LambdaSubscriber 属于:Subscriber的实现类
```
### (6).Flux.subscribeWith
```
public final <E extends Subscriber<? super T>> E subscribeWith(E subscriber) {
    // reactor.core.publisher.LambdaSubscriber
    // 调用最终的subscribe
    subscribe(subscriber);
    return subscriber;
}
```
### (7).Flux.subscribe
```
public final void subscribe(Subscriber<? super T> actual) {
    // actual = reactor.core.publisher.LambdaSubscriber
    
    // ***************************************************
    // 把FluxArray转换成:CorePublisher
    // this = FluxArray
    // ***************************************************
    CorePublisher publisher = Operators.onLastAssembly(this);
    
    // ***************************************************
    // 把reactor.core.publisher.LambdaSubscriber
    // 转换成: CoreSubscriber
    // ***************************************************
    CoreSubscriber subscriber = Operators.toCoreSubscriber(actual);

    try {
        // 从上面的分析得了:publisher不是OptimizableOperator的子类
        if (publisher instanceof OptimizableOperator) {  // false
            OptimizableOperator operator = (OptimizableOperator) publisher;
            while (true) {
                subscriber = operator.subscribeOrReturn(subscriber);
                if (subscriber == null) {
                    return;
                }
                
                OptimizableOperator newSource = operator.nextOptimizableSource();
                
                if (newSource == null) {
                    publisher = operator.source();
						break;
                } //end if
                
                operator = newSource;
            } //end while
        } //end if
        
        
        // ***************************************************
        // reactor.core.publisher.FluxArray
        //   .subscribe(reactor.core.publisher.LambdaSubscriber s)
        // ***************************************************
        publisher.subscribe(subscriber);
    } catch (Throwable e) {
        Operators.reportThrowInSubscribe(subscriber, e);
        return;
    }
}
```
### (8).FluxArray.subscribe
```
public void subscribe(CoreSubscriber<? super T> actual) {
    // actual = reactor.core.publisher.LambdaSubscriber
    // array = ["1","2"]
    subscribe(actual, array);
}

public static <T> void subscribe(CoreSubscriber<? super T> s, T[] array) {
        // s = reactor.core.publisher.LambdaSubscriber
        // array = ["1","2"]
        if (array.length == 0) {  // false
            Operators.complete(s);
            return;
        }// end if
        
        if (s instanceof ConditionalSubscriber) { // false
            s.onSubscribe(new ArrayConditionalSubscription<>((ConditionalSubscriber<? super T>) s, array));
        } else {
            // *********************Subscriber.onSubscribe*************************
            // reactor.core.publisher.LambdaSubscriber.onSubscribe
            // 
            // **********************************************
            s.onSubscribe(new ArraySubscription<>(s, array));
        } //end else
}
```
### (9).LambdaSubscriber.onSubscribe
```
// reactor.core.publisher.FluxArray$ArraySubscription
public final void onSubscribe(Subscription s) {
    // s = reactor.core.publisher.FluxArray$ArraySubscription
    // s包裹着:数组和LambdaSubscriber
    if (Operators.validate(subscription, s)) { // true
            // 订阅者
            this.subscription = s;
            
            if (subscriptionConsumer != null) { // false
                try {
                    subscriptionConsumer.accept(s);
                } catch (Throwable t) {
                    Exceptions.throwIfFatal(t);
                    s.cancel();
                    onError(t);
                }
            } else {
                // s = FluxArray$ArraySubscription
                s.request(Long.MAX_VALUE);
            } //end else
    } //end if
}
```
### (10).FluxArray.ArraySubscription
```
public void request(long n) {
    // Long.MAX_VALUE
    if (Operators.validate(n)) { // true
            if (Operators.addCap(REQUESTED, this, n) == 0) { // true
                if (n == Long.MAX_VALUE) { // true
                   // invoker fastPath
                    fastPath();
                } else {
                    slowPath(n);
                }
            } //end if
    } //end if
}
```
### (11).FluxArray.ArraySubscription.fastPath
```
void fastPath() {
    // a = [1, 2]
    final T[] a = array;
    // len = 2
    final int len = a.length;
    // s = reactor.core.publisher.LambdaSubscriber
    final Subscriber<? super T> s = actual;

    // 遍历数组
    for (int i = index; i != len; i++) {
        // 是否设置了取消
        if (cancelled) {
                return;
        }

        // 数组第N个元素
        T t = a[i];

        
        if (t == null) {
            // 为空的情况下调用:onError
            s.onError(new NullPointerException("The " + i + "th array element was null"));
            return;
        }
        // ************************************
        // onNext调用
        // ************************************
        // 最后调用onNext方法
        s.onNext(t);
    }// end for
        
    if (cancelled) {
            return;
    } //end if
        
    // 最后调用:onCompelete()方法    
    s.onComplete();
}
```
### (12).总结
1. Flux.just()方法.  
    会把传入的数据封装成:FluxArray(它属于Publisher的子类)
2. Flux.subscribe(Consumer c)方法.  
    会把Consumer函数封装成:LambdaSubscriber(它属于Subscriber的子类)
3. Flux.subscribe(Subscriber) 最后实际是这样调用的:  
    FluxArray.subscribe(LambdaSubscriber s)
4. 把数据和消费者封装成:ArraySubscription(属于:Subscription的子类)  
    LambdaSubscriber
        .onSubscribe(
            new ArraySubscription(LambdaSubscriber,
            ["1","2"])
    );  
    注意:ArraySubscription持有:LambdaSubscriber和数组
5. 设置一次订阅多少  
    ArraySubscription.request(Long.MAX_VALUE) 
6. 进行快速消费或缓存消费  
   T[] data = ["1","2"];
    for(int i=0;i<data.length;i++){
        如果数组元素有一个为空:
        则调用:ArraySubscription.onError(...)
        否则调用:ArraySubscription.onNext(...)
    }
    ArraySubscription.onComplete()