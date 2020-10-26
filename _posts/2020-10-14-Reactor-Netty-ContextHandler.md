---
layout: post
title: 'Project Reactor Netty源码(ContextHandler)'
date: 2020-07-21
author: 李新
tags: ReactorNetty
---

### (1).发起请求
> 用户需要发起请求,才会触发:ChannelInitializer的调用
### (2).ContextHandler继承关系
```
public abstract class ContextHandler<CHANNEL extends Channel>
		extends ChannelInitializer<CHANNEL> implements Disposable, Consumer<Channel> {
}
// ContextHandler继承于:ChannelInitializer,所以,必须要实现:initChannel()方法
// ContextHandler 实现了:Consumer<Channel>

```
### (3).ContextHandler.initChannel
```
protected void initChannel(CHANNEL ch) throws Exception {
    // 委托给了:accept方法
    accept(ch);
}

```
### (4).ContextHandler.accept
```
public void accept(Channel channel) {
    // 4. 委托给子类(ServerContextHandler.doPipeline)
    doPipeline(channel);
    if (options.onChannelInit() != null) { // false
        if (options.onChannelInit().test(channel)) {
                if (log.isDebugEnabled()) {
                    log.debug("DROPPED by onChannelInit predicate {}", channel);
                }
                doDropped(channel);
                return;
        }
    }

    try {
        // 6. HttpServer.TcpBridgeServer
        if (pipelineConfigurator != null) { //true
               // HttpServer.TcpBridgeServer.accept()
                pipelineConfigurator.accept(channel.pipeline(), (ContextHandler<Channel>) this);
        }
        
        // reactor.ipc.netty.channel.ChannelOperationsHandler
        channel.pipeline().addLast(NettyPipeline.ReactiveBridge,new ChannelOperationsHandler(this));
    } catch (Exception t) {
        if (log.isErrorEnabled()) {
                log.error("Error while binding a channelOperation with: " + channel.toString() + " on " + channel.pipeline(), t);
        }
    } finally {
            if (null != options.afterChannelInit()) { // false
                options.afterChannelInit().accept(channel);
            }
    }
    
    if (log.isDebugEnabled()) {
            log.debug("After pipeline {}", channel.pipeline().toString());
    }
}
```
### (5).ServerContextHandler.doPipeline
```
protected void doPipeline(Channel ch) {
    // 调用父类模板方法(ContextHandler.addSslAndLogHandlers)
    addSslAndLogHandlers(options, this, loggingHandler, true, getSNI(), ch.pipeline());
}
```
### (6).ContextHandler.addSslAndLogHandlers
```
static void addSslAndLogHandlers(NettyOptions<?, ?> options,
			ContextHandler<?> sink,
			LoggingHandler loggingHandler,
			boolean secure,
			Tuple2<String, Integer> sniInfo,
			ChannelPipeline pipeline) {
    // secure=true   
    SslHandler sslHandler = secure ? options.getSslHandler(pipeline.channel().alloc(), sniInfo) : null;
    // sslHandler = null
    if (sslHandler != null) { //false
        if (log.isDebugEnabled() && sniInfo != null) {
                log.debug("SSL enabled using engine {} and SNI {}",
        		sslHandler.engine()
		          .getClass()
		          .getSimpleName(),
    		   sniInfo);
        } else if (log.isDebugEnabled()) {
            log.debug("SSL enabled using engine {}",
    		sslHandler.engine()
		          .getClass()
		          .getSimpleName());
        } 
        if (log.isTraceEnabled()) {
            pipeline.addFirst(NettyPipeline.SslLoggingHandler, new LoggingHandler(SslReadHandler.class));
            pipeline.addAfter(NettyPipeline.SslLoggingHandler, NettyPipeline.SslHandler, sslHandler);
        } else {
            pipeline.addFirst(NettyPipeline.SslHandler, sslHandler);
        } // else 
        
        if (log.isDebugEnabled()) {
                pipeline.addAfter(NettyPipeline.SslHandler, NettyPipeline.LoggingHandler,loggingHandler);
                pipeline.addAfter(NettyPipeline.LoggingHandler,NettyPipeline.SslReader,new SslReadHandler(sink));
        } else {
                pipeline.addAfter(NettyPipeline.SslHandler,
    	    NettyPipeline.SslReader,new SslReadHandler(sink));
        }
    } else if (log.isDebugEnabled()) { //true
        // 如果开启了debug,则添加日志处理
        // io.netty.handler.logging.LoggingHandler
        pipeline.addFirst(NettyPipeline.LoggingHandler, loggingHandler);
    } //end else if
} //end addSslAndLogHandlers
```
### (7).HttpServer.TcpBridgeServer.accept
```
public void accept(ChannelPipeline p, ContextHandler<Channel> c) {
    // 创建HttpServerCodec编码解码器
    // io.netty.handler.codec.http.HttpServerCodec
    p.addLast(NettyPipeline.HttpCodec, new HttpServerCodec(
            	    options.httpCodecMaxInitialLineLength(),  // 4096
		            options.httpCodecMaxHeaderSize(),         // 8192
		            options.httpCodecMaxChunkSize(),          // 8192
		            options.httpCodecValidateHeaders(),       // true
		            options.httpCodecInitialBufferSize()));   // 128 
     // gzip压缩是否开启
    if(options.minCompressionResponseSize() >= 0) { // false
        // reactor.ipc.netty.http.server.CompressionHandler
        p.addLast(NettyPipeline.CompressionHandler, new CompressionHandler(options.minCompressionResponseSize()));
    }
    // **************************************************
    // 业务处理代码了:
    // reactor.ipc.netty.http.server.HttpServerHandler
    // **************************************************
    p.addLast(NettyPipeline.HttpServerHandler, new HttpServerHandler(c));
}
```
### (8).所有过滤链
```
io.netty.handler.logging.LoggingHandler
io.netty.handler.codec.http.HttpServerCodec
reactor.ipc.netty.http.server.CompressionHandler
reactor.ipc.netty.http.server.HttpServerHandler
reactor.ipc.netty.channel.ChannelOperationsHandler
```
### (9).总结
> 最终触发ChannelInitializer的调用