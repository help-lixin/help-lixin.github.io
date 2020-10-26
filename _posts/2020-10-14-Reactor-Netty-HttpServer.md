---
layout: post
title: 'Project Reactor Netty源码(HttpServer)'
date: 2020-07-14
author: 李新
tags: ReactorNetty
---

### (1).HttpServer
> 基于Netty实现提供Http服务类
### (2).HttpServer
```
HttpServer.create(8080)  // 创建HttpServer并指定端口为8080
```
### (3).HttpServer.Builder
> 通过HttpServer.Builder配置监听IP和端口信息,并通过build方法构建出:HttpServer对象

```
public final class HttpServer
             // INBOUND :     HttpServerRequest
             // OUTBOUND :    HttpServerResponse
             implements NettyConnector<HttpServerRequest, HttpServerResponse> {
     
    // 1.创建指定端口的:HttpServer 
    public static HttpServer create(int port) {  // 8080
        return 
                // 1.1 创建HttpServer.Builder对象
                builder()
                // 1.3 设置监听端口
               .listenAddress(new InetSocketAddress(port))
               // 1.4 构建:HttpServer
              .build();
	} //end create
 
   // 1.2 创建:HttpServer.Builder
    public static HttpServer.Builder builder() {
        return new HttpServer.Builder();
	}
 
    // HttpServer.Builder
    public static final class Builder {
        // 
        private String bindAddress = null;
        private int port = 8080;
        private Supplier<InetSocketAddress> listenAddress = () -> new InetSocketAddress(NetUtil.LOCALHOST, port);
        private Consumer<? super HttpServerOptions.Builder> options;

        private Builder() {
        }
        
        // 1.3 设置监听端口
        public final Builder listenAddress(InetSocketAddress listenAddress) {
            Objects.requireNonNull(listenAddress, "listenAddress");
            this.listenAddress = () -> listenAddress;
            return this;
        }
        
        // *************************************************
        // 1.4 构建:HttpServer
        // 通过HttpServer.Builder配置参数
        // *************************************************
        public HttpServer build() {
            return new HttpServer(this);
        }
        
    } //end Builder
    
    
  } //end HttpServer
```
### (4).HttpServer构造器
```
public final class HttpServer
		implements NettyConnector<HttpServerRequest, HttpServerResponse> {
    
    // TcpBrigeServer
    private final TcpBridgeServer server;
    // HttpServer的配置参数
	final HttpServerOptions options;
    
    // 1.构建HttpServer
    private HttpServer(HttpServer.Builder builder) {
        // 1.1 构建出一个临时的:HttpServerOptions.Builder
        // ********************************************
        // HttpServerOptions是一个高级别的配置参数 
        // ********************************************
        HttpServerOptions.Builder serverOptionsBuilder = HttpServerOptions.builder();
        // 如果HttpServer.Builder.options中有配置:HttpServerOptions.Builder
        if (Objects.isNull(builder.options)) { // true
                if (Objects.isNull(builder.bindAddress)) { // true
                        // 为serverOptionsBuilder设置监听端口(0.0.0.0/0.0.0.0:8080)
                        serverOptionsBuilder.listenAddress(builder.listenAddress.get());
                } else {
                        serverOptionsBuilder.host(builder.bindAddress).port(builder.port);
                }
        } else { 
            builder.options.accept(serverOptionsBuilder);
        } //end else
        
        if (!serverOptionsBuilder.isLoopAvailable()) { // (!false)
            // *********************************************
            // 先忽略,后面用一章节来讲解
            // *********************************************
            serverOptionsBuilder.loopResources(HttpResources.get());
        } //end if
        
        // 1.2 serverOptionsBuilder构建出可选参数
        this.options = serverOptionsBuilder.build();
        this.server = new TcpBridgeServer(this.options);
	}//end HttpServer(HttpServer.Builder builder)
    
}
```
### (5).总结
1. HttpServer.Builder配置参数(监听端口信息等)  
2. HttpServer.Builder.buid() 构造出一个:HttpServer  
3. 读取HttpServer.Builder 并转换到:HttpServerOptions.Builder对象里  
4. 创建TcpBridgeServer  