---
layout: post
title: 'Project Reactor Netty源码(TcpBridgeServer)'
date: 2020-07-14
author: 李新
tags: ReactorNetty
---

### (1).TcpBridgeServer
```
public final class HttpServer
		implements NettyConnector<HttpServerRequest, HttpServerResponse> {

    private final TcpBridgeServer server;
    final HttpServerOptions options;
     
    private HttpServer(HttpServer.Builder builder) {
        // ... ... 
        // 1.  创建Tcp桥接服务,传递:HttpServerOptions
       this.server = new TcpBridgeServer(this.options);
       // ... ... 
    } //end HttpServer
        
}
```
### (2).TcpBridgeServer继承关系
```
class HttpServer.TcpBridgeServer 
      extends TcpServer
      implements BiConsumer<ChannelPipeline, ContextHandler<Channel>> {}
      
class TcpServer implements NettyConnector<NettyInbound, NettyOutbound> {}

interface NettyConnector<INBOUND extends NettyInbound, OUTBOUND extends NettyOutbound> {}

// HttpServer.TcpBridgeServer 继承于:TcpServer 同时实现了:BiConsumer.accept(T,U)
```
### (3).HttpServer.TcpBridgeServer
```
public final class HttpServer
		implements NettyConnector<HttpServerRequest, HttpServerResponse> {
    
    
    final class TcpBridgeServer 
          extends TcpServer
          implements BiConsumer<ChannelPipeline, ContextHandler<Channel>> {
              
            // 1.创建:TcpBridgeServer          
           TcpBridgeServer(ServerOptions options) {
               // 委托给父类
                super(options);
    	} //end TcpBridgeServer构建器
          
    } //end HttpServer.TcpBridgeServer
    
} //end HttpServer
```
### (4).TcpServer
```
public class TcpServer implements NettyConnector<NettyInbound, NettyOutbound> {
    // 2.设置:ServerOptions
    protected TcpServer(ServerOptions options) {
        this.options = Objects.requireNonNull(options, "options");
	} //end TcpServer 构建器
} //end TcpServer
```