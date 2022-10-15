---
layout: post
title: 'Zeebe ClusterServicesStep源码之NettyMessagingService初始化之Server(五)' 
date: 2022-10-14
author: 李新
tags:  Zeebe
---

### (1). 概述
在前面的剖析,发现NettyMessagingService的构造器,主要是创建:ChannelPool,并没有什么特殊的东西,我们接着往下剖析,看下:NettyMessagingService内部到底干了什么? 

### (2). NettyMessagingService.start
```
public CompletableFuture<MessagingService> start() {
     // ... ... 
	 
	 // 初始化:EventLoopGroup
    initTransport();
	
	
    return serviceLoader
	     // ************************************************************* 
		 // 启动Server
		 // ************************************************************* 
        .thenCompose(ok -> bootstrapServer())
        .thenRun(
            () -> {
              timeoutExecutor =
                  Executors.newSingleThreadScheduledExecutor(
                      new DefaultThreadFactory("netty-messaging-timeout-"));
              localConnection = new LocalClientConnection(handlers);
              started.set(true);

              log.info(
                  "Started messaging service bound to {}, advertising {}, and using {}",
                  bindingAddresses,
                  advertisedAddress,
                  config.isTlsEnabled() ? "TLS" : "plaintext");
            })
        .thenApply(v -> this);
} // end


private void initTransport() {
	if (Epoll.isAvailable()) {
	  initEpollTransport();
	} else {
	  initNioTransport();
	}
} // end 


private void initEpollTransport() {
    clientGroup = new EpollEventLoopGroup(0, namedThreads("netty-messaging-event-epoll-client-%d", log));
    serverGroup = new EpollEventLoopGroup(0, namedThreads("netty-messaging-event-epoll-server-%d", log));
    serverChannelClass = EpollServerSocketChannel.class;
    clientChannelClass = EpollSocketChannel.class;
} //end 

private void initNioTransport() {
    clientGroup = new NioEventLoopGroup(0, namedThreads("netty-messaging-event-nio-client-%d", log));
    serverGroup = new NioEventLoopGroup(0, namedThreads("netty-messaging-event-nio-server-%d", log));
    serverChannelClass = NioServerSocketChannel.class;
    clientChannelClass = NioSocketChannel.class;
}// end 
```
### (3). NettyMessagingService.bootstrapServer
```
private CompletableFuture<Void> bootstrapServer() {
    final ServerBootstrap b = new ServerBootstrap();
    b.option(ChannelOption.SO_REUSEADDR, true);
    b.option(ChannelOption.SO_BACKLOG, 128);
    b.childOption(
        ChannelOption.WRITE_BUFFER_WATER_MARK, new WriteBufferWaterMark(8 * 1024, 32 * 1024));
    b.childOption(ChannelOption.SO_RCVBUF, 1024 * 1024);
    b.childOption(ChannelOption.SO_SNDBUF, 1024 * 1024);
    b.childOption(ChannelOption.SO_KEEPALIVE, true);
    b.childOption(ChannelOption.TCP_NODELAY, true);
    b.childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
    b.group(serverGroup, clientGroup);
    b.channel(serverChannelClass);
	// ************************************************************************
	// 配置ChannelInitializer
	// ************************************************************************
    b.childHandler(new BasicServerChannelInitializer());
	
	
    return bind(b);
}
```

### (4). NettyMessagingService$BasicServerChannelInitializer

```
private class BasicServerChannelInitializer extends ChannelInitializer<SocketChannel> {

    @Override
    protected void initChannel(final SocketChannel channel) {
      // ... ...
      // *************************************************************************
	  // 添加了握手协议
	  // *************************************************************************
      channel.pipeline().addLast("handshake", new ServerHandshakeHandlerAdapter());

      // ... ...
    } //end 
}
```

### (5). NettyMessagingService$ServerHandshakeHandlerAdapter
```
private class ServerHandshakeHandlerAdapter extends HandshakeHandlerAdapter<ProtocolRequest> {

    @Override
    public void channelRead(final ChannelHandlerContext context, final Object message)
        throws Exception {
      // Read the protocol version from the client handshake. If the client's protocol version is
      // unknown
      // to the server, use the latest server protocol version.
      readProtocolVersion(context, (ByteBuf) message)
          .ifPresent(
              version -> {
                // ... ...
				// ***************************************************************
				// 
				// ***************************************************************
                activateProtocolVersion(
                    context,
                    new RemoteServerConnection(handlers, context.channel()),
                    protocolVersion);
              });
    }

    @Override
    void activateProtocolVersion(
        final ChannelHandlerContext context,
        final Connection<ProtocolRequest> connection,
        final ProtocolVersion protocolVersion) {
        // ... ... 
		// ***************************************************************
		// 
		// ***************************************************************
      super.activateProtocolVersion(context, connection, protocolVersion);
    }
  }
```
### (6). NettyMessagingService.activateProtocolVersion
```
void activateProtocolVersion(
        final ChannelHandlerContext context,
        final Connection<M> connection,
        final ProtocolVersion protocolVersion) {
	  // *****************************************************************		
	  // 创建消息编码与解码处理器
	  // *****************************************************************		
      final MessagingProtocol protocol = protocolVersion.createProtocol(advertisedAddress);
	  // 从Pipeline移除当前的编解码器(NettyMessagingService$ServerHandshakeHandlerAdapter)
      context.pipeline().remove(this);
	  
      context.pipeline().addLast("encoder", protocol.newEncoder());
      context.pipeline().addLast("decoder", protocol.newDecoder());
      context.pipeline().addLast("handler", new MessageDispatcher<>(connection));
}
```
### (8). 总结
NettyMessagingService的底层实际上是通过Netty监听一个端口(26502),通过这两个类进行编码和解码(MessageToByteEncoder/ByteToMessageDecoder),最终,会把请求移交给:MessageDispatcher进行处理. 
