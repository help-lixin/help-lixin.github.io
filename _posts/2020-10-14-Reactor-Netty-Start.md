---
layout: post
title: 'Project Reactor Netty源码(HttpServer.startRouter)'
date: 2020-07-14
author: 李新
tags: ReactorNetty
---

### (1).HttpServer.startRouter
```
public final class HttpServer
		implements NettyConnector<HttpServerRequest, HttpServerResponse> {

    
    public BlockingNettyContext startRouter(Consumer<? super HttpServerRoutes> routesBuilder) {
        Objects.requireNonNull(routesBuilder, "routeBuilder");
        // 构建路由信息久类
        HttpServerRoutes routes = HttpServerRoutes.newRoutes();
        // 回调给:routesBuilder处理,看上一章
        routesBuilder.accept(routes);
        // ***********************************************
        // 1.启动,调用父类方法(NettyConnector.start)
        // ***********************************************
        return start(routes);
	}// end startRouter
}
```
### (2).NettyConnector.start
```
default <T extends BiFunction<INBOUND, OUTBOUND, ? extends Publisher<Void>>>
    // 2.start方法
    BlockingNettyContext start(T handler) {
        return new BlockingNettyContext(
                        //调用子类:HttpServer.newHandler()
                        newHandler(handler), 
                        getClass().getSimpleName()
                   );
    } // end start
```
### (3).HttpServer.newHandler
```
public Mono<? extends NettyContext> newHandler(BiFunction<? super HttpServerRequest, ? super HttpServerResponse, ? extends Publisher<Void>> handler) {
        Objects.requireNonNull(handler, "handler");
        // 把DefaultHttpServerRoutes强制转换成(BiFunction<HttpServerRequest, HttpServerResponse, Publisher<Void>>)        
        // 3.委托给:TcpBridgeServer.newHandler
        // TcpBridgeServer继承于:TcpServer.newHandler
        return server.newHandler((BiFunction<NettyInbound, NettyOutbound, Publisher<Void>>) handler);
}
```
### (4).TcpServer.newHandler
```
public final Mono<? extends NettyContext> newHandler(BiFunction<? super NettyInbound, ? super NettyOutbound, ? extends Publisher<Void>> handler) {
    Objects.requireNonNull(handler, "handler");
    return Mono.create(sink -> {
        ServerBootstrap b = options.get();
        SocketAddress local = options.getAddress();
        b.localAddress(local);
        // 4.调用子类:HttpServer.TcpBridgeServer.doHandler(handler, sink)
        ContextHandler<Channel> contextHandler = doHandler(handler, sink);
        b.childHandler(contextHandler);
        if(log.isDebugEnabled()){
            b.handler(loggingHandler());
        }
        contextHandler.setFuture(b.bind());
    });
}
```
### (5).HttpServer.TcpBridgeServer.doHandler
```
protected ContextHandler<Channel> doHandler(
        BiFunction<? super NettyInbound, ? super NettyOutbound, ? extends Publisher<Void>> handler,
        MonoSink<NettyContext> sink) {
        // 5.调用:ContextHandler.newServerContext
        return ContextHandler.newServerContext(
                              sink,
                              options,
                              loggingHandler,
                              (ch, c, msg) -> HttpServerOperations.bindHttp(ch, handler, c, msg)
            )
            .onPipeline(this)
            .autoCreateOperations(false);
}
```
### (6).ContextHandler.newServerContext
```
// ContextHandler<Channel> 继承了Netty的:ChannelInitializer
public static ContextHandler<Channel> newServerContext(
        MonoSink<NettyContext> sink,
        ServerOptions options,
        LoggingHandler loggingHandler,
        ChannelOperations.OnNew<Channel> channelOpFactory) {
    // 6.创建ServerContextHandler(属于:ContextHandler的子类)
    return new ServerContextHandler(channelOpFactory, options, sink, loggingHandler, options.getAddress());
}
```
### (7).总结
1. TcpServer.newHandler获得:ServerBootstrap  
2. 调用HttpServer.TcpBridgeServer.doHandler()创建:ContextHandler(ChannelInitializer)  
3. 创建:ServerContextHandler(ChannelInitializer)  
4. ServerContextHandler实现了ChannelInitializer.initChannel方法  
5. 在Netty启动时,ChannelInitializer.initChannel不会调用,而是要等到第一次请求时才会发生调用.  