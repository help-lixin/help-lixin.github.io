---
layout: post
title: 'Project Reactor Netty源码(NettyConnector)'
date: 2020-07-21
author: 李新
tags: ReactorNetty
---

### (1).NettyConnector
> NettyConnector提供了基本的网络连接接口,其实现类有:  
HttpServer/TcpServer/UdpServer...
在这里只讨论:HttpServer

### (2).NettyConnector接口声明
```
package reactor.ipc.netty;

public interface NettyConnector
                 // 定义泛型,NettyInbound/NettyOutbound
                 <INBOUND extends NettyInbound, 
                 OUTBOUND extends NettyOutbound> {
    
    // 创建一个Handler
    Mono<? extends NettyContext> newHandler(  
        // 入参:INBOUND/OUTBOUND  出参:Publisher<Void>
        // public interface BiFunction<T, U, R> { R apply(T t, U u)  }
        BiFunction<? super INBOUND, ? super OUTBOUND, ? extends Publisher<Void>> ioHandler
    );
 
   // 
    default <T extends BiFunction<INBOUND, OUTBOUND, ? extends Publisher<Void>>>
        BlockingNettyContext start(T handler) {
            // 最终还是调用了抽象子类的方法 
            return new BlockingNettyContext(newHandler(handler), getClass().getSimpleName());
        }
 
    // 
    default <T extends BiFunction<INBOUND, OUTBOUND, ? extends Publisher<Void>>>
        BlockingNettyContext start(T handler, Duration timeout) {
            return new BlockingNettyContext(newHandler(handler), getClass().getSimpleName(), timeout);
        }

    //Publisher<Void>
    default <T extends BiFunction<INBOUND, OUTBOUND, ? extends Publisher<Void>>>
    	void startAndAwait(T handler, @Nullable Consumer<BlockingNettyContext> onStart) {
                 BlockingNettyContext facade = new BlockingNettyContext(newHandler(handler), getClass().getSimpleName());
                 facade.installShutdownHook();
		      if (onStart != null) {
			   onStart.accept(facade);
		       }
		      facade.getContext().onClose().block();
	}
}
```
### (3).总结
> 从接口申明可以看出:NettyConnector提供了所有网络请求的抽象.里面只有一个抽象方法newHandler,参数类型为:NettyInbound/NettyOutbound/出参为:Publisher<Void>
