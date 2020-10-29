---
layout: post
title: 'Spring Cloud Stream源码(DirectWithAttributesChannel)'
date: 2020-07-03
author: 李新
tags: SpringCloudStream
---

### (1).ProvderService
```
@Component
public class ProvderService {
	
	@Autowired
	private Source source;
	
	public void sned(String message) {
        //  *****************************************************
        // 1. 发送消息
       // source被BindableProxyFactory进行了代理
       // 最终source = DirectWithAttributesChannel
       //  *****************************************************
        source.output().send(MessageBuilder.withPayload(message).build());
	}
}
```
### (2).DirectWithAttributesChannel
```
public abstract class AbstractMessageChannel 
                extends IntegrationObjectSupport
		    implements MessageChannel, 
                           TrackableComponent, 
                           ChannelInterceptorAware, 
                           MessageChannelMetrics,
                           ConfigurableMetricsAware<AbstractMessageChannelMetrics> {
    
     // *******************************************************  
    //  2. 发送消息
    // *******************************************************  
    public boolean send(Message<?> message, long timeout) {
        Assert.notNull(message, "message must not be null");
        Assert.notNull(message.getPayload(), "message payload must not be null");
        if (this.shouldTrack) { // false
            message = MessageHistory.write(message, this, this.getMessageBuilderFactory());
        }
        
        Deque<ChannelInterceptor> interceptorStack = null;
        boolean sent = false;
        boolean metricsProcessed = false;
        MetricsContext metrics = null;
        boolean countsEnabled = this.countsEnabled; // true
        
        // 通道拦截器
        // org.springframework.integration.channel.AbstractMessageChannel$ChannelInterceptorList
        ChannelInterceptorList interceptors = this.interceptors;
        AbstractMessageChannelMetrics channelMetrics = this.channelMetrics;
        SampleFacade sample = null;
        try {
            if (this.datatypes.length > 0) { // false
                message = this.convertPayloadIfNecessary(message);
            }
            
            //  false
            boolean debugEnabled = this.loggingEnabled && logger.isDebugEnabled();
            if (debugEnabled) { // false
                logger.debug("preSend on channel '" + this + "', message: " + message);
            }
            
            // true
            if (interceptors.getSize() > 0) {
                interceptorStack = new ArrayDeque<>();
                // ********************************************************
                // 3. 调用拦截器的:preSend(.....)
                // interceptorStack = [org.springframework.cloud.stream.binding.MessageConverterConfigurer$OutboundContentTypeConvertingInterceptor@6d815fb7]
                // ********************************************************
                message = interceptors.preSend(message, this, interceptorStack);
                if (message == null) { // false
                   // 发送失败
                    return false;
                }
            }
            
            if (countsEnabled) { // true
                metrics = channelMetrics.beforeSend();
                
                if (this.metricsCaptor != null) { // true
                    sample = this.metricsCaptor.start();
                }
                
                // *******************************************************
                // 委托给子类发送消息
                // 4. AbstractSubscribableChannel.doSend(...)
                // *******************************************************
                sent = doSend(message, timeout);
                
                if (sample != null) {
                    	sample.stop(sendTimer(sent));
                }
                channelMetrics.afterSend(metrics, sent);
                metricsProcessed = true;
            } else { // false
                sent = doSend(message, timeout);
            }

            if (debugEnabled) { // false
                logger.debug("postSend (sent=" + sent + ") on channel '" + this + "', message: " + message);
            }
            
            
            if (interceptorStack != null) {
                
                // *****************************************************
                // 调用拦截器
                //     postSend
                //     afterSendCompletion
                // *****************************************************
                interceptors.postSend(message, this, sent);
                interceptors.afterSendCompletion(message, this, sent, null, interceptorStack);
            }
            
            return sent;
        } catch (Exception e) {
            if (countsEnabled && !metricsProcessed) {
                if (sample != null) {
                    sample.stop(buildSendTimer(false, e.getClass().getSimpleName()));
                }
                channelMetrics.afterSend(metrics, false);
            }
            
            if (interceptorStack != null) {
				interceptors.afterSendCompletion(message, this, sent, e, interceptorStack);
            }
            throw IntegrationUtils.wrapInDeliveryExceptionIfNecessary(message,
					() -> "failed to send Message to channel '" + this.getComponentName() + "'", e);
        }
	}// end send      
 
 
     protected static class ChannelInterceptorList {
         // ************************************************
         // 3.1 发送消息之前
        // ************************************************
         public Message<?> preSend(
                           Message<?> message, 
                           MessageChannel channel,
				Deque<ChannelInterceptor> interceptorStack) {
            if (this.size > 0) {
                // 遍历所有的拦截器
                for (ChannelInterceptor interceptor : this.interceptors) {
                        Message<?> previous = message;
                        // 挨个执行:preSend
                        // ************************************************
                        // 3.2 preSend(....)
                        // ************************************************
                        message = interceptor.preSend(message, channel);
                        if (message == null) { // false
                            if (this.logger.isDebugEnabled()) {
                                this.logger.debug(interceptor.getClass().getSimpleName() + " returned null from preSend, i.e. precluding the send.");
                            } //end if
                            afterSendCompletion(previous, channel, false, null, interceptorStack);
                            return null;
                        } //end if

                        // 记录执行过有哪些拦截器
                        interceptorStack.add(interceptor);
                }// end for
            }
            return message;
		} // end preSend
     }
 
      
  }
```
### (3).AbstractSubscribableChannel
```
public abstract class AbstractSubscribableChannel extends AbstractMessageChannel
		implements SubscribableChannel, SubscribableChannelManagement {

    // ************************************************************
    // 4.1 发送消息
    // ************************************************************
    protected boolean doSend(Message<?> message, long timeout) {
        try {
             //  *********************************************
            // 委托给消息分发器进行分发消息: UnicastingDispatcher
            // *********************************************
            return getRequiredDispatcher().dispatch(message);
        } catch (MessageDispatchingException e) {
                String description = e.getMessage() + " for channel '" + this.getFullChannelName() + "'.";
                throw new MessageDeliveryException(message, description, e);
        } // end try
	}// end doSend
 
    // 4.2 获得消息分发器 
    private MessageDispatcher getRequiredDispatcher() {
        // 委托给子类:DirectChannel 获得消息分发器
        MessageDispatcher dispatcher = getDispatcher();
        Assert.state(dispatcher != null, "'dispatcher' must not be null");
        return dispatcher;
	}// end getRequiredDispatcher

  }
```
### (4).DirectChannel
```
public class DirectChannel extends AbstractSubscribableChannel {
    private final UnicastingDispatcher dispatcher = new UnicastingDispatcher();
    
    // 4.3 获得消息分发器
    protected UnicastingDispatcher getDispatcher() {
		return this.dispatcher;
	}
}
```
### (5).MessageConverterConfigurer$OutboundContentTypeConvertingInterceptor
```
public class MessageConverterConfigurer implements MessageChannelAndSourceConfigurer, BeanFactoryAware {
 
    private final class OutboundContentTypeConvertingInterceptor extends AbstractContentTypeInterceptor {
        private final MessageConverter messageConverter;

        private OutboundContentTypeConvertingInterceptor(String contentType, CompositeMessageConverter messageConverter) {
            super(contentType);
            this.messageConverter = messageConverter;
        }


        // *********************************************************************
        // 3.3 doPreSend(....)
        // *********************************************************************
        @Override
        public Message<?> doPreSend(Message<?> message, MessageChannel channel) {
            // ===== 1.3 backward compatibility code part-1 ===
            // 判断协议头是否有:contentType
            // oct = null
            String oct = message.getHeaders().containsKey(MessageHeaders.CONTENT_TYPE) ? message.getHeaders().get(MessageHeaders.CONTENT_TYPE).toString() : null;
            // ctx = text/plain
            String ct = message.getPayload() instanceof String
						? ct = JavaClassMimeTypeUtils.mimeTypeFromObject(message.getPayload(), ObjectUtils.nullSafeToString(oct)).toString()
						: oct;
            // ===== END 1.3 backward compatibility code part-1 ===

            // true
            if (!message.getHeaders().containsKey(MessageHeaders.CONTENT_TYPE)) {
                @SuppressWarnings("unchecked")
                Map<String, Object> headersMap = (Map<String, Object>) ReflectionUtils.getField(MessageConverterConfigurer.this.headersField, message.getHeaders());
                // 设置:contentType=application/json
                headersMap.put(MessageHeaders.CONTENT_TYPE, this.mimeType);
            }

            @SuppressWarnings("unchecked")
            // 委托给:CompositeMessageConverter进行消息转换
            // 循环调用以下MessageConverter.
            // [org.springframework.cloud.stream.converter.ApplicationJsonMessageMarshallingConverter@376af784, org.springframework.cloud.stream.converter.TupleJsonMessageConverter@a9a8ec3, org.springframework.cloud.stream.converter.CompositeMessageConverterFactory$1@6d0114c0, org.springframework.cloud.stream.converter.ObjectStringMessageConverter@40016ce1, org.springframework.cloud.stream.converter.JavaSerializationMessageConverter@203765b2, org.springframework.cloud.stream.converter.KryoMessageConverter@7e050be1, org.springframework.cloud.stream.converter.JsonUnmarshallingConverter@1b47b7f5]
            Message<byte[]> outboundMessage = message.getPayload() instanceof byte[]
					? (Message<byte[]>)message : (Message<byte[]>) this.messageConverter.toMessage(message.getPayload(), message.getHeaders());
            
            if (outboundMessage == null) { // false
                throw new IllegalStateException("Failed to convert message: '" + message + "' to outbound message.");
            } // end if

            /// ===== 1.3 backward compatibility code part-2 ===
            if (ct != null && !ct.equals(oct) && oct != null) { // false
                @SuppressWarnings("unchecked")
                Map<String, Object> headersMap = (Map<String, Object>) ReflectionUtils.getField(MessageConverterConfigurer.this.headersField, outboundMessage.getHeaders());
                headersMap.put(MessageHeaders.CONTENT_TYPE, MimeType.valueOf(ct));
                headersMap.put(BinderHeaders.BINDER_ORIGINAL_CONTENT_TYPE, MimeType.valueOf(oct));
            } //end if
            // ===== END 1.3 backward compatibility code part-2 ===
            return outboundMessage;
        } //end doPreSend
	}// end OutboundContentTypeConvertingInterceptor
          
}
```
### (6).总结
> 1. 调用:source.output().send(...);
> 2. BindableProxyFactory拦截请求,委托给:DirectWithAttributesChannel进行发送
> 3. 发送之前先调用拦截器的:ChannelInterceptorList.preSend()方法,对消息进行编码
> 4. 然后委托给子类:DirectChannel.doSend(),DirectChannel会委托给:UnicastingDispatcher进行消息请求的分发(**UnicastingDispatcher参考另一篇文章**).
> 5. 消息发送完之后,会调用拦截器的:ChannelInterceptorList.postSend/afterSendCompletion.
