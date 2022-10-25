---
layout: post
title: 'Zeebe ClusterServicesStep源码之RemoteServerConnection(十三)' 
date: 2022-10-14
author: 李新
tags:  Zeebe
---

### (1). 概述
在前面对NettyMessagingService进行了简单的了解,它主要的职责是进行通信的管理,对于Server(服务端)来说,底层原理应该是要接受请求,并回调给业务代码来着,所以,我们要找到NettyMessagingService与Netty的交接处:ChannelInboundHandlerAdapter.
### (2). NettyMessagingService$HandshakeHandlerAdapter
```
public class NettyMessagingService {
	
	// ************************************************************************************
	// 1. Zeebe定义,握手处理抽象类.
	// ************************************************************************************
	private abstract class HandshakeHandlerAdapter<M extends ProtocolMessage> extends ChannelInboundHandlerAdapter {
		void writeProtocolVersion(final ChannelHandlerContext context, final ProtocolVersion version) {
			 // ... ... 
		} // end 
		
		OptionalInt readProtocolVersion(final ChannelHandlerContext context, final ByteBuf buffer) {
			// ... ...
		} // end  
		
		void activateProtocolVersion(
		        final ChannelHandlerContext context,
		        final Connection<M> connection,
		        final ProtocolVersion protocolVersion) {
		    final MessagingProtocol protocol = protocolVersion.createProtocol(advertisedAddress);
            context.pipeline().remove(this);
			
            context.pipeline().addLast("encoder", protocol.newEncoder());
            context.pipeline().addLast("decoder", protocol.newDecoder());
            context.pipeline().addLast("handler", new MessageDispatcher<>(connection));
		} // end activateProtocolVersion
	} // end HandshakeHandlerAdapter
} // end NettyMessagingService
```
### (3). NettyMessagingService$ServerHandshakeHandlerAdapter
```
public class NettyMessagingService {
	
	private class ServerHandshakeHandlerAdapter extends HandshakeHandlerAdapter<ProtocolRequest> {
		@Override
		    public void channelRead(final ChannelHandlerContext context, final Object message)
		        throws Exception {
		      readProtocolVersion(context, (ByteBuf) message)
		          .ifPresent(
		              version -> {
		                ProtocolVersion protocolVersion = ProtocolVersion.valueOf(version);
		                if (protocolVersion == null) {
		                  protocolVersion = ProtocolVersion.latest();
		                }
						
		                writeProtocolVersion(context, protocolVersion);
						
		                activateProtocolVersion(
		                    context,
							// ****************************************************************
							// 创建了一个RemoteServerConnection
							// ****************************************************************
		                    new RemoteServerConnection(handlers, context.channel()),
		                    protocolVersion);
		              });
		    } // end channelRead
		
		    @Override
		    void activateProtocolVersion(
		        final ChannelHandlerContext context,
		        final Connection<ProtocolRequest> connection,
		        final ProtocolVersion protocolVersion) {
		      log.debug(
		          "Activating server protocol version {} for connection to {}",
		          protocolVersion,
		          context.channel().remoteAddress());
		      super.activateProtocolVersion(context, connection, protocolVersion);
		    } // end activateProtocolVersion
	} // end NettyMessagingService$ServerHandshakeHandlerAdapter
}
```
### (4). RemoteServerConnection
> RemoteServerConnection是我们的核心类了,它相当于一个桥梁类,一侧与Netty打交道,另一侧与我们的业务Handler打交道. 

```
interface Connection<M extends ProtocolMessage> {
	// 分发消息
	void dispatch(M message);
	// ... ... 
} // end 

interface ServerConnection extends Connection<ProtocolRequest> {
	// 回复消息
	void reply(ProtocolRequest message, ProtocolReply.Status status, Optional<byte[]> payload);
	// ... ... 
} // end 

abstract class AbstractServerConnection implements ServerConnection {
	// *****************************************************************
	// HandlerRegistry是一个典型的中介者模式,下一篇剖析跟它相关的内容.
	// *****************************************************************
	private final HandlerRegistry handlers;
	
	AbstractServerConnection(final HandlerRegistry handlers) {
		this.handlers = handlers;
	} // end AbstractServerConnection
	
	
	public void dispatch(final ProtocolRequest message) {
	    final String subject = message.subject();
		// *****************************************************************
		// 2.  根据subject,从HandlerRegistry中获取一个:BiConsumer<ProtocolRequest, ServerConnection>处理消息
		// *****************************************************************
	    final BiConsumer<ProtocolRequest, ServerConnection> handler = handlers.get(subject);
	    if (handler != null) {
	      log.trace("Received message type {} from {}", subject, message.sender());
	      handler.accept(message, this);
	    } else {
          // *****************************************************************
		  // 3. 未找到Handler处理方式.
		  // *****************************************************************
	      log.debug("No handler for message type {} from {}", subject, message.sender());
	      byte[] subjectBytes = null;
	      if (subject != null) {
	        subjectBytes = StringUtil.getBytes(subject);
	      }
	
	      // *****************************************************************
	      //  4. reply交给子类(RemoteServerConnection)处理.
		  //     因为,自己都是一个抽象类
		  // *****************************************************************
	      reply(message, ProtocolReply.Status.ERROR_NO_HANDLER, Optional.ofNullable(subjectBytes));
	    } 
	} // end dispatch
} // end 

final class RemoteServerConnection extends AbstractServerConnection {
	RemoteServerConnection(final HandlerRegistry handlers, final Channel channel) {
	    super(handlers);
	    this.channel = channel;
	} // end RemoteServerConnection
	
	public void reply(
	      final ProtocolRequest message,
	      final ProtocolReply.Status status,
	      final Optional<byte[]> payload) {
		// 仅仅是通过Netty Channel写出了一个协议消息.	  
	    final ProtocolReply response = new ProtocolReply(message.id(), payload.orElse(EMPTY_PAYLOAD), status);
	    channel.writeAndFlush(response, channel.voidPromise());
	} // end reply
	
} // end 	
```
### (5). 总结
NettyMessagingService底层,最终是委派给:RemoteServerConnection进行通信,而,RemoteServerConnection负责分发请求给:HandlerRegistry.  