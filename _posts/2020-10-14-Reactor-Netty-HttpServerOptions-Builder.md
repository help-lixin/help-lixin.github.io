---
layout: post
title: 'Project Reactor Netty源码(HttpServerOptions.Builder)'
date: 2020-07-21
author: 李新
tags: ReactorNetty
---

### (1).HttpServerOptions.Builder.loopResources 
```
private HttpServer(HttpServer.Builder builder) {
    // *********************************************
    // 构建HttpServer配置信息
    // *********************************************
    HttpServerOptions.Builder serverOptionsBuilder =
                                   // new ServerBootstrap() 
                                   HttpServerOptions.builder();
    // .... ...
    if (!serverOptionsBuilder.isLoopAvailable()) {
            // 1. loopResources实际为:创建:netty ServerBootstrap
            serverOptionsBuilder.loopResources(HttpResources.get());
    }
    // ****************************************************
    // 6.构建:HttpServerOptions
    // ****************************************************
    this.options = serverOptionsBuilder.build();
    // .... ...
}
```
### (2).HttpServerOptions继承关系
```
public abstract class NettyOptions<BOOTSTRAP 
                extends AbstractBootstrap<BOOTSTRAP, ?>, SO extends NettyOptions<BOOTSTRAP, SO>>
		    implements Supplier<BOOTSTRAP>{}


public class ServerOptions extends NettyOptions<ServerBootstrap, ServerOptions> {}      

public final class HttpServerOptions extends ServerOptions {}

// HttpServerOptions 继承于 ServerOptions
// ServerOptions 继承于:NettyOptions
// NettyOptions 是一个抽象类
```
### (3).NettyOptions.Builder
```
public abstract class NettyOptions<BOOTSTRAP extends AbstractBootstrap<BOOTSTRAP, ?>, SO extends NettyOptions<BOOTSTRAP, SO>>
		implements Supplier<BOOTSTRAP> {
      
    // 11.构建:NettyOptions  
    protected NettyOptions(NettyOptions.Builder<BOOTSTRAP, SO, ?> builder) {
		this.bootstrapTemplate = builder.bootstrapTemplate;
		this.preferNative = builder.preferNative;
		this.loopResources = builder.loopResources;
		this.sslContext = builder.sslContext;
		this.sslHandshakeTimeoutMillis = builder.sslHandshakeTimeoutMillis;
		this.sslCloseNotifyFlushTimeoutMillis = builder.sslCloseNotifyFlushTimeoutMillis;
		this.sslCloseNotifyReadTimeoutMillis = builder.sslCloseNotifyReadTimeoutMillis;
		this.afterNettyContextInit = builder.afterNettyContextInit;
		this.onChannelInit = builder.onChannelInit;

		Consumer<? super Channel> afterChannel = builder.afterChannelInit;
		if (afterChannel != null && builder.channelGroup != null) {
			this.afterChannelInit = ((Consumer<Channel>) builder.channelGroup::add)
					.andThen(afterChannel);
		} else if (afterChannel != null) {
			this.afterChannelInit = afterChannel;
		} else if (builder.channelGroup != null) {
			this.afterChannelInit = builder.channelGroup::add;
		} else {
			this.afterChannelInit = null;
		}
	} //end NettyOptions
      
      
    public static abstract class Builder<BOOTSTRAP extends AbstractBootstrap<BOOTSTRAP, ?>,
			SO extends NettyOptions<BOOTSTRAP, SO>, BUILDER extends Builder<BOOTSTRAP, SO, BUILDER>>
			implements Supplier<BUILDER> {

        protected Builder(BOOTSTRAP bootstrapTemplate) {
                this.bootstrapTemplate = bootstrapTemplate;
                // ***************************************
                // 配置Netty配置信息
                // ***************************************
                defaultNettyOptions(this.bootstrapTemplate);
        } //end Builder
        
        private void defaultNettyOptions(AbstractBootstrap<?, ?> bootstrap) {
                bootstrap.option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
        } //end defaultNettyOptions

        // 2.loopResources加载资源
       public final BUILDER loopResources(LoopResources channelResources) {
            this.loopResources = Objects.requireNonNull(channelResources, "loopResources");
            // 3. 委托给子类:ServerOptions.Build
            return get();
        }// end loopResources
        
   } //end NettyOptions.Builder
}
```
### (4).ServerOptions.Builder
```
public class ServerOptions extends NettyOptions<ServerBootstrap, ServerOptions> {
    
    
    public static class Builder<BUILDER extends Builder<BUILDER>>
			extends NettyOptions.Builder<ServerBootstrap, ServerOptions, BUILDER> {
       
        protected Builder(ServerBootstrap serverBootstrap) {
                super(serverBootstrap);
                // *******************************************
                // 为ServerBootstrap配置默认值
                // *******************************************
                defaultServerOptions(serverBootstrap);
        } // end Builder
        
        private final void defaultServerOptions(ServerBootstrap bootstrap) {
                bootstrap.option(ChannelOption.SO_REUSEADDR, true)
                    .option(ChannelOption.SO_BACKLOG, 1000)
                    .childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
                    .childOption(ChannelOption.SO_RCVBUF, 1024 * 1024)
                    .childOption(ChannelOption.SO_SNDBUF, 1024 * 1024)
                    .childOption(ChannelOption.AUTO_READ, false)
                    .childOption(ChannelOption.SO_KEEPALIVE, true)
                    .childOption(ChannelOption.TCP_NODELAY, true)
                    .childOption(ChannelOption.CONNECT_TIMEOUT_MILLIS, 30000);
        } // end defaultServerOptions
        
        // 4.返回:HttpServer.Build对象
        public BUILDER get() {
            // reactor.ipc.netty.http.server.HttpServerOptions$Builder
            return (BUILDER) this;
        } //end get
        
        // 9.构建:ServerOptions
        public ServerOptions build() {
            // 10. 构建:NettyOptions
            // 12. 构建ServerOptions
            return new ServerOptions(this);
        } //end build
        
    } // end ServerOptios.Build
    
    // 12. 构建ServerOptions
    protected ServerOptions(ServerOptions.Builder<?> builder) {
		super(builder); // 10. 构建:NettyOptions
		if (Objects.isNull(builder.listenAddress)) {
                if (Objects.isNull(builder.host)) {
                    this.localAddress = new InetSocketAddress(builder.port);
                } else {
                    this.localAddress = InetSocketAddressUtil.createResolved(builder.host, builder.port);
                }
		} else {
                    this.localAddress = builder.listenAddress instanceof InetSocketAddress
					? InetSocketAddressUtil.replaceWithResolved((InetSocketAddress) builder.listenAddress): builder.listenAddress;
		}
	}// end ServerOptions
}
```
### (5).HttpServerOptions.Builder
```
public final class HttpServerOptions extends ServerOptions {
    
    
    public static final class Builder extends ServerOptions.Builder<Builder> {
        // 内部实则为构建:ServerBootstrap
        private Builder(){
            super(new ServerBootstrap());
        }
        
        // 7. 构建:HttpServerOptions
        public HttpServerOptions build() {
            // 8. 委托给父类:ServerOptions.Builder
            super.build();
            return new HttpServerOptions(this);
        } // end build
        
    } //end HttpServerOptions.Builder
    
    // 13.构建:HttpServerOptions
    private HttpServerOptions(HttpServerOptions.Builder builder) {
		super(builder);
		this.minCompressionResponseSize = builder.minCompressionResponseSize;
		this.maxInitialLineLength = builder.maxInitialLineLength;
		this.maxHeaderSize = builder.maxHeaderSize;
		this.maxChunkSize = builder.maxChunkSize;
		this.validateHeaders = builder.validateHeaders;
		this.initialBufferSize = builder.initialBufferSize;
	} //end HttpServerOptions
}
```
### (6).总结
1. HttpServerOptions.builder实则为,创建:ServerBootstrap.
2. HttpServerOptions.loopResources实则为:ServerBootstrap配置NioEventLoopGroup