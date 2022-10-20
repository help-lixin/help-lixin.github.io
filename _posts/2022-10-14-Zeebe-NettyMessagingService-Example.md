---
layout: post
title: 'Zeebe ClusterServicesStep源码之NettyMessagingService案例(十二)' 
date: 2022-10-14
author: 李新
tags:  Zeebe
---

### (1). 概述
在前面的源码中,稍有剖析NettyMessagingService,它是底层的通信基石,所以,这一小篇,通过一个小小的案例,来了解这个类的功能点.

### (2). NettyMessagingServiceCompressionTest
```
package io.atomix.cluster.messaging.impl;

import static org.assertj.core.api.Assertions.assertThat;

import io.atomix.cluster.messaging.ManagedMessagingService;
import io.atomix.cluster.messaging.MessagingConfig;
import io.atomix.cluster.messaging.MessagingConfig.CompressionAlgorithm;
import io.atomix.utils.net.Address;
import io.camunda.zeebe.test.util.socket.SocketUtil;
import java.time.Duration;
import java.util.concurrent.CompletableFuture;

class NettyMessagingServiceCompressionTest {
  
  void shouldSendAndReceiveMessagesWhenCompressionEnabled() {
    // 1.配置压缩算法
    final var config =
        new MessagingConfig()
            .setShutdownQuietPeriod(Duration.ofMillis(50))
            .setCompressionAlgorithm(CompressionAlgorithm.NONE);
    
	// 2. 监听1025端口,并启动(主要是用于发送消息)
    final var senderAddress = Address.from(SocketUtil.getNextAddress().getPort());
    final var senderNetty = (ManagedMessagingService)new NettyMessagingService("test", senderAddress, config).start().join();

	// 3. 监听1026端口,并启动(主要用于接收消息).
    final var receiverAddress = Address.from(SocketUtil.getNextAddress().getPort());
    final var receiverNetty = (ManagedMessagingService)new NettyMessagingService("test", receiverAddress, config).start().join();

    // 定义主题
    final String subject = "subject";
	// 定义发送消息的消息体
    final String requestString = "message";
	// 定义回复消息的消息体
    final String responseString = "success";
	
	// 4. 配置处理器,监听某个主题(subject),并,回复消息
    receiverNetty.registerHandler(
        subject,
        (m, payload) -> {
          final String message = new String(payload);
          assertThat(message).isEqualTo(requestString);
          return CompletableFuture.completedFuture(responseString.getBytes());
        });

    // 5. 向某个IP:PORT发送消息.
    final CompletableFuture<byte[]> response = senderNetty.sendAndReceive(receiverAddress, subject, requestString.getBytes());

    // 6. 等待结果返回
    final var result = response.join();
    assertThat(new String(result)).isEqualTo(responseString);

    // 7. 关闭
    senderNetty.stop();
    receiverNetty.stop();
  }
}
```
### (3). 总结
之所以,拿出这样一个案例是因为,看了下Zeebe针对Netty通信方面的的源码写得还是挺不错的,后面想把代码抠出来,复用下. 