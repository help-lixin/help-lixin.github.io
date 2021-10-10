---
layout: post
title: '查看Spring Integration相关业务模型能力(三)' 
date: 2021-10-09
author: 李新
tags:  SpringIntegration
---

### (1). 概述
在这一小篇,主要了解一下,SI抽象了哪些接口.

### (2). Message
Message是SI对消息载体的抽象,一般是由MessageBuilder创建.   

```
package org.springframework.messaging;

// 通过MessageBuilder构建出一个消息.
// org.springframework.messaging.support.MessageBuilder
public interface Message<T> {

	// 消息体
	T getPayload();
	
	// 消息头
	MessageHeaders getHeaders();
}
```
### (3). MessageChannel
MessageChannel是SI对"发送消息"能力的一个抽象.

```
package org.springframework.messaging;

public interface MessageChannel {
	
	long INDEFINITE_TIMEOUT = -1;
	
	// 发送消息
	boolean send(Message<?> message);
	
	// 发送消息,并定义发送消息的超时时间
	boolean send(Message<?> message, long timeout);
}
```
### (4). SubscribableChannel
SubscribableChannel是SI"接受消息能力"的抽象.  

```
package org.springframework.messaging;


public interface SubscribableChannel extends MessageChannel {

	/**
	 * 绑定消息的处理
	 * Register a message handler.
	 * @return {@code true} if the handler was subscribed or {@code false} if it
	 * was already subscribed.
	 */
	boolean subscribe(MessageHandler handler);

	/**
	 * 解绑消息的处理
	 * Un-register a message handler.
	 * @return {@code true} if the handler was un-registered, or {@code false}
	 * if was not registered.
	 */
	boolean unsubscribe(MessageHandler handler);
}
```
### (5). MessageHandler
MessageHandler一看就知道是"消息的处理"抽象.

```
package org.springframework.messaging;

// 消息的处理
public interface MessageHandler {

	/**
	 * 消息的处理模型
	 * Handle the given message.
	 * @param message the message to be handled
	 * @throws MessagingException if the handler failed to process the message
	 */
	void handleMessage(Message<?> message) throws MessagingException;

}
```