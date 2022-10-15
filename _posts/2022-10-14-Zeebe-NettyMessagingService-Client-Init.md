---
layout: post
title: 'Zeebe ClusterServicesStep源码之NettyMessagingService初始化之Client(四)' 
date: 2022-10-14
author: 李新
tags:  Zeebe
---

### (1). 概述
在前面对StartupStep进行了剖析,这一小篇主要剖析:ClusterServicesStep中的一小部份,即:AtomixCluster创建过程的详解.

### (2). ClusterServicesStep.startupInternal
```
void startupInternal(
      final BrokerStartupContext brokerStartupContext,
      final ConcurrencyControl concurrencyControl,
      final ActorFuture<BrokerStartupContext> startupFuture) {
	
	// ... ... 
	// ***********************************************************************
	// 运用静态工厂模式,构建:AtomixCluster
	// ***********************************************************************
	final var atomix =
	        AtomixClusterFactory.fromConfiguration(brokerStartupContext.getBrokerConfiguration());
	// ... ... 
}
```
### (3). AtomixClusterFactory.fromConfiguration
```
public static AtomixCluster fromConfiguration(final BrokerCfg configuration) {
	// ... ... 
	var atomixBuilder  = new AtomixClusterBuilder(new ClusterConfig());
	
	// ... ...
	return atomixBuilder.build();
}	
```
### (4). AtomixCluster.build
```
public AtomixCluster build() {
	return new AtomixCluster(config, Version.from(VersionUtil.getVersion()));
}
```
### (5). AtomixCluster构建器
```
public AtomixCluster(final ClusterConfig config, final Version version) {
	// *****************************************************************
	// 在这一篇,主要剖析:buildMessagingService方法
	//   下一篇,再剖析:buildUnicastService方法
	// *****************************************************************
    this(config, version, buildMessagingService(config), buildUnicastService(config));
}
```
### (6). AtomixCluster.buildMessagingService
```
protected static ManagedMessagingService buildMessagingService(final ClusterConfig config) {
	// config.getNodeConfig().getAddress() = 26502
    return new NettyMessagingService(config.getClusterId(), config.getNodeConfig().getAddress(), config.getMessagingConfig());
}
```
### (7). NettyMessagingService构建器
```
public NettyMessagingService(
      final String cluster, final Address advertisedAddress, final MessagingConfig config) {
    // 调用下面这个要构造器的方法.
    this(cluster, advertisedAddress, config, ProtocolVersion.latest());
}

NettyMessagingService(
  final String cluster,
  final Address advertisedAddress,
  final MessagingConfig config,
  final ProtocolVersion protocolVersion) {
	preamble = cluster.hashCode();
	this.advertisedAddress = advertisedAddress;
	this.protocolVersion = protocolVersion;
	this.config = config;

	openFutures = new CopyOnWriteArrayList<>();
	// *************************************************************************
	// ChannelPool看这个类名称,大概就知道应该是与Netty里的Channel相关,并且是一个池子.
	// *************************************************************************
	channelPool = new ChannelPool(this::openChannel, config.getConnectionPoolSize());
	// ... ...
}
```
### (8). NettyMessagingService.openChannel
```
private CompletableFuture<Channel> openChannel(final Address address) {
	return bootstrapClient(address);
}
```
### (9). NettyMessagingService.bootstrapClient
> 从这个名称上来看,是client端的Channel,这个client的Channel要等待到发送请求(部署流程或其它)时才会调用一次.  

```
private CompletableFuture<Channel> bootstrapClient(final Address address) {
	// address:0.0.0.0:26501
    final CompletableFuture<Channel> future = new OrderedFuture<>();
    final InetAddress resolvedAddress = address.address(true);
    if (resolvedAddress == null) {
      future.completeExceptionally(new IllegalStateException("Failed to bootstrap client (address " + address.toString() + " cannot be resolved)"));
      return future;
    }

    final Bootstrap bootstrap = new Bootstrap();
    bootstrap.option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
    bootstrap.option( ChannelOption.WRITE_BUFFER_WATER_MARK,
        new WriteBufferWaterMark(10 * 32 * 1024, 10 * 64 * 1024));
    bootstrap.option(ChannelOption.SO_RCVBUF, 1024 * 1024);
    bootstrap.option(ChannelOption.SO_SNDBUF, 1024 * 1024);
    bootstrap.option(ChannelOption.SO_KEEPALIVE, true);
    bootstrap.option(ChannelOption.TCP_NODELAY, true);
    bootstrap.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 1000);
    bootstrap.group(clientGroup);
    bootstrap.channel(clientChannelClass);
	// 0.0.0.0:26501
    bootstrap.remoteAddress(resolvedAddress, address.port());
	// ********************************************************************
	// Client Channel初始化配置
	// ********************************************************************
    bootstrap.handler(new BasicClientChannelInitializer(future));
    final Channel channel =
        bootstrap
            .connect()
            .addListener(
                onConnect -> {
                  if (!onConnect.isSuccess()) {
                    future.completeExceptionally(
                        new ConnectException(
                            String.format("Failed to connect channel for address %s", address)));
                  }
                })
            .channel();

    
    channel
        .closeFuture()
        .addListener(
            onClose ->
                future.completeExceptionally(
                    new ConnectException(
                        String.format(
                            "Channel %s for address %s was closed unexpectedly before the request was handled",
                            channel, address))));

    return future;
}
```
### (10). NettyMessagingService$BasicClientChannelInitializer
```
private class BasicClientChannelInitializer extends ChannelInitializer<SocketChannel> {
	@Override
	protected void initChannel(final SocketChannel channel) {
	  // ... ... 
      // *********************************************************************
	  // 握手协议
	  // *********************************************************************
	  channel.pipeline().addLast("handshake", new ClientHandshakeHandlerAdapter(future));
	  // ... ... 
	}
}
```
### (11). NettyMessagingService$ClientHandshakeHandlerAdapter
```
private class ClientHandshakeHandlerAdapter extends HandshakeHandlerAdapter<ProtocolReply> {

    private final CompletableFuture<Channel> future;

    ClientHandshakeHandlerAdapter(final CompletableFuture<Channel> future) {
      this.future = future;
    }

    @Override
    public void channelActive(final ChannelHandlerContext context) throws Exception {
      log.debug(
          "Writing client protocol version {} for connection to {}",
          protocolVersion,
          context.channel().remoteAddress());
	  // ***************************************************************
	  // 当client Channel初始化状态时,发送出一个握手协议信息.
	  // ***************************************************************
      writeProtocolVersion(context, protocolVersion);
    }
	
	// 写出一个协议信息
	// void writeProtocolVersion(final ChannelHandlerContext context, final ProtocolVersion version) {
	//      final ByteBuf buffer = context.alloc().buffer(6);
	//      buffer.writeInt(preamble);
	//      buffer.writeShort(version.version());
	//      context.writeAndFlush(buffer);
	// }
	

    @Override
    public void channelRead(final ChannelHandlerContext context, final Object message)
        throws Exception {
	   // 读取服务端的协议信息	
      // Read the protocol version from the server.
      readProtocolVersion(context, (ByteBuf) message)
          .ifPresent(
              version -> {
				 // 先确保协议信息是正确的.
                final ProtocolVersion protocolVersion = ProtocolVersion.valueOf(version);
                if (protocolVersion != null) {
				  // **************************************************
				  // 创建连接,并配置:pipeline
				  // **************************************************
                  activateProtocolVersion(context, getOrCreateClientConnection(context.channel()), protocolVersion);
                } else {
                  log.error("Failed to negotiate protocol version");
                  context.close();
                }
              });
    }
}
```
### (12). NettyMessagingService.activateProtocolVersion
```
void activateProtocolVersion(
        final ChannelHandlerContext context,
        final Connection<M> connection,
        final ProtocolVersion protocolVersion) {
	  // **************************************************************************		
	  // 重点:编码和解码
	  // **************************************************************************		
      final MessagingProtocol protocol = protocolVersion.createProtocol(advertisedAddress);
	  // 从当前的Pipeline中移除编解码顺(NettyMessagingService$ClientHandshakeHandlerAdapter)
      context.pipeline().remove(this);
      context.pipeline().addLast("encoder", protocol.newEncoder());
      context.pipeline().addLast("decoder", protocol.newDecoder());
      context.pipeline().addLast("handler", new MessageDispatcher<>(connection));
}
```
### (13). NettyMessagingService.getOrCreateClientConnection
```
// ... ...
private final Map<Channel, RemoteClientConnection> connections = Maps.newConcurrentMap();
// ... ...

private RemoteClientConnection getOrCreateClientConnection(final Channel channel) {
    RemoteClientConnection connection = connections.get(channel);
    if (connection == null) {
	  // ***********************************************************************
	  //  通过:RemoteClientConnection包裹着:Channel(Client)
	  // ***********************************************************************
      connection = connections.computeIfAbsent(channel, RemoteClientConnection::new);
      channel
          .closeFuture()
          .addListener(
              f -> {
                final RemoteClientConnection removedConnection = connections.remove(channel);
                if (removedConnection != null) {
                  removedConnection.close();
                }
              });
    }
    return connection;
}
```
### (14). ChannelPool
```
class ChannelPool {
	// 从方法签名上就能看出来:是根据address创建:Channel
	CompletableFuture<Channel> getChannel(final Address address, final String messageType);
	
}
```
### (15). 总结
分析了大半天,NettyMessagingService的构造器在初始化时,最重要的是创建了:ChannelPool,从名称上就能看出来,它是一个Netty中Channel的池子.  