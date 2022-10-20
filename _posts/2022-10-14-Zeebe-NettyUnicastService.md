---
layout: post
title: 'Zeebe ClusterServicesStep源码之NettyUnicastService(六)' 
date: 2022-10-14
author: 李新
tags:  Zeebe
---

### (1). 概述
在前面分析NettyMessagingService的构建器时,会看到有构建一个类:NettyUnicastService,那这个类主要是干嘛呢?

### (2). AtomixCluster构建器
```
public AtomixCluster(final ClusterConfig config, final Version version) {
	// *****************************************************
	// buildUnicastService
	// *****************************************************
    this(config, version, buildMessagingService(config), buildUnicastService(config));
}
```
### (3). AtomixCluster.buildUnicastService
```
protected static ManagedUnicastService buildUnicastService(final ClusterConfig config) {
	// 26502
    return new NettyUnicastService(config.getClusterId(), config.getNodeConfig().getAddress(), config.getMessagingConfig());
}
```
### (4). ManagedUnicastService
> 先看下接口

```
public interface ManagedUnicastService extends UnicastService, Managed<UnicastService> {}


// 广播消息
public interface UnicastService {
	// 广播
	void unicast(Address address, String subject, byte[] message);
	// 对接收的消息,添加监听器进行处理
	void addListener(String subject, BiConsumer<Address, byte[]> listener, Executor executor);
	// 移除监听器
	void removeListener(String subject, BiConsumer<Address, byte[]> listener);
}	
```
### (5). NettyUnicastService.start
```
public CompletableFuture<UnicastService> start() {
    group = new NioEventLoopGroup(0, namedThreads("netty-unicast-event-nio-client-%d", log));
	// *****************************************************************
	// 启动
	// *****************************************************************
    return bootstrap().thenRun(() -> started.set(true)).thenApply(v -> this);
}
```
### (6). NettyUnicastService.bootstrap
```
private CompletableFuture<Void> bootstrap() {
    final Bootstrap serverBootstrap =
        new Bootstrap()
            .group(group)
			// ************************************************************
			// NioDatagramChannel
			// UDP通信
			// ************************************************************
            .channel(NioDatagramChannel.class)
            .handler(
                new SimpleChannelInboundHandler<DatagramPacket>() {
                  @Override
                  protected void channelRead0(
                      final ChannelHandlerContext context, final DatagramPacket packet)
                      throws Exception {
					// ************************************************************
					// 接收并处理消息
					// ************************************************************
                    handleReceivedPacket(packet);
                  }
                })
            .option(ChannelOption.RCVBUF_ALLOCATOR, new DefaultMaxBytesRecvByteBufAllocator())
            .option(ChannelOption.SO_BROADCAST, true)
            .option(ChannelOption.SO_REUSEADDR, true);

    return bind(serverBootstrap);
}
```
### (7). NettyUnicastService.handleReceivedPacket
```
private void handleReceivedPacket(final DatagramPacket packet) {
	// 集群名称的hashcode值
    final int preambleReceived = packet.content().readInt();
    if (preambleReceived != preamble) { // 集群名称不同时,忽悠这个消息
      log.warn(
          "Received unicast message from {} which is outside of the cluster. Ignoring the message.",
          packet.sender());
      return;
    }
	
	// 读取body的字节数
    final byte[] payload = new byte[packet.content().readInt()];
    packet.content().readBytes(payload);
	
	//  通过:Kryo解码消息
    final Message message = SERIALIZER.decode(payload);
	//  主题监听器
    final Map<BiConsumer<Address, byte[]>, Executor> subjectListeners = listeners.get(message.subject());
    if (subjectListeners != null) {
       //  遍历所有的listener,进行消息的处理.
      subjectListeners.forEach(
          (consumer, executor) ->
              executor.execute(() -> consumer.accept(message.source(), message.payload())));
    }
} // end 
```
### (8). NettyUnicastService.unicast
```
public void unicast(final Address address, final String subject, final byte[] payload) {
    if (!started.get()) {
      LOGGER.debug("Failed sending unicast message, unicast service was not started.");
      return;
    }

    final InetAddress resolvedAddress = address.address();
    if (resolvedAddress == null) {
      LOGGER.debug(
          "Failed sending unicast message (destination address {} cannot be resolved)", address);
      return;
    }
    
	// 通过Kryo进行编码.
    final Message message = new Message(this.address, subject, payload);
	
    final byte[] bytes = SERIALIZER.encode(message);
	// ************************************************************
	// 4 + 4 + N
	// 4 : preamble(集群的hashcode)
	// 4 : playbody bytes length
	// ************************************************************
    final ByteBuf buf = channel.alloc().buffer(Integer.BYTES + Integer.BYTES + bytes.length);
    buf.writeInt(preamble);
    buf.writeInt(bytes.length).writeBytes(bytes);
	// 直接往指定的address进行消息发送
    channel.writeAndFlush(
        new DatagramPacket(buf, new InetSocketAddress(resolvedAddress, address.port())));
} // end 
```
### (9). NettyUnicastServiceTest
```
package io.atomix.cluster.messaging.impl;

import static org.assertj.core.api.Assertions.assertThatCode;
import static org.junit.Assert.assertArrayEquals;
import static org.junit.Assert.assertEquals;
import static org.slf4j.LoggerFactory.getLogger;

import io.atomix.cluster.messaging.ManagedUnicastService;
import io.atomix.cluster.messaging.MessagingConfig;
import io.atomix.utils.net.Address;
import io.camunda.zeebe.test.util.socket.SocketUtil;
import net.jodah.concurrentunit.ConcurrentTestCase;
import org.junit.After;
import org.junit.Before;
import org.junit.Test;
import org.slf4j.Logger;

public class NettyUnicastServiceTest extends ConcurrentTestCase {

  private static final Logger LOGGER = getLogger(NettyUnicastServiceTest.class);

  ManagedUnicastService service1;
  ManagedUnicastService service2;

  Address address1;    // 127.0.0.1:1025
  Address address2;    // 127.0.0.1:1025

  @Before
  public void setUp() throws Exception {
    address1 = Address.from("127.0.0.1", SocketUtil.getNextAddress().getPort());
    address2 = Address.from("127.0.0.1", SocketUtil.getNextAddress().getPort());

    final String clusterId = "testClusterId";
    service1 = new NettyUnicastService(clusterId, address1, new MessagingConfig());
    service1.start().join();

    service2 = new NettyUnicastService(clusterId, address2, new MessagingConfig());
    service2.start().join();
  } // end setUp
  
  @Test
  public void testUnicast() throws Exception {
  	// 监听的:1025端口,添加主题(test)处理器.
    service1.addListener(
        "test",
        (address, payload) -> {
          assertEquals(address2, address);
          assertArrayEquals("Hello world!".getBytes(), payload);
          resume();
        });
    
    // 1026 --> 1025 发送广播消息
    service2.unicast(address1, "test", "Hello world!".getBytes());
    await(5000);
  } // end testUnicast

  @After
  public void tearDown() throws Exception {
    if (service1 != null) {
      try {
        service1.stop().join();
      } catch (final Exception e) {
        LOGGER.warn("Failed stopping netty1", e);
      }
    }

    if (service2 != null) {
      try {
        service2.stop().join();
      } catch (final Exception e) {
        LOGGER.warn("Failed stopping netty2", e);
      }
    }
  } // end tearDown
}
```
### (10). 总结
NettyMessagingService的底层是,在启动时监听UDP端口(26502),主要用于广播和监听消息来着的.  