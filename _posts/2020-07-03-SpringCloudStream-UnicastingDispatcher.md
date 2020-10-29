---
layout: post
title: 'Spring Cloud Stream源码(UnicastingDispatcher)'
date: 2020-07-03
author: 李新
tags: SpringCloudStream
---

### (1).叙述
> 前面说过:DirectChannel发委托给:UnicastingDispatcher.dispatch进行请求的分发.

### (2).UnicastingDispatcher
```
public class UnicastingDispatcher extends AbstractDispatcher {
    // ************************************************
    // 1.分发消息
    // ************************************************
    public final boolean dispatch(final Message<?> message) {
        // 如果有配置执行器,就交给执行器进行管理
        if (this.executor != null) { // false
            Runnable task = createMessageHandlingTask(message);
            this.executor.execute(task);
            return true;
        }// end if
        
       
       return this.doDispatch(message);
	}// end dispatch    
 
    private boolean doDispatch(Message<?> message) {
        // ************************************************
        // 2. 委托给父类,判断消息是否需要优化分发
        // ************************************************
        if (tryOptimizedDispatch(message)) {
            return true;
        }
        
        boolean success = false;
        Iterator<MessageHandler> handlerIterator = this.getHandlerIterator(message);
        if (!handlerIterator.hasNext()) {
            throw new MessageDispatchingException(message, "Dispatcher has no subscribers");
        }
        
        List<RuntimeException> exceptions = new ArrayList<RuntimeException>();
        while (!success && handlerIterator.hasNext()) {
            MessageHandler handler = handlerIterator.next();
            try {
                handler.handleMessage(message);
                success = true; // we have a winner.
            } catch (Exception e) {
                RuntimeException runtimeException = IntegrationUtils.wrapInDeliveryExceptionIfNecessary(message, () -> "Dispatcher failed to deliver Message", e);
                exceptions.add(runtimeException);
                this.handleExceptions(exceptions, message, !handlerIterator.hasNext());
            } //end catch
        }
        return success;
    }// end doDispatch
 
}
```
### (3).AbstractDispatcher
```
public abstract class AbstractDispatcher implements MessageDispatcher {
    // ************************************************************
    // 3. 优化消息分发
    // ************************************************************
    protected boolean tryOptimizedDispatch(Message<?> message) {
        MessageHandler handler = this.theOneHandler;
        // org.springframework.cloud.stream.binder.AbstractMessageChannelBinder$SendingHandler
        if (handler != null) {
            try {
                // 委托给:org.springframework.cloud.stream.binder.AbstractMessageChannelBinder$SendingHandler
                // 进行消息处理
                handler.handleMessage(message);
                return true;
            } catch (Exception e) {
                throw IntegrationUtils.wrapInDeliveryExceptionIfNecessary(message, () -> "Dispatcher failed to deliver Message", e);
            }
        } //end if
        return false;
	} //end tryOptimizedDispatch
}

```
### (4).AbstractMessageHandler
```
public abstract class AbstractMessageHandler extends IntegrationObjectSupport
		implements MessageHandler, MessageHandlerMetrics, ConfigurableMetricsAware<AbstractMessageHandlerMetrics>,
		TrackableComponent, Orderable, CoreSubscriber<Message<?>> {

    // **************************************************
    //  4. 处理消息
    // **************************************************
    public void handleMessage(Message<?> message) {
		Assert.notNull(message, "Message must not be null");
		Assert.notNull(message.getPayload(), "Message payload must not be null"); //NOSONAR - false positive
		if (this.loggingEnabled && this.logger.isDebugEnabled()) {
			this.logger.debug(this + " received message: " + message);
		}
		MetricsContext start = null;
		boolean countsEnabled = this.countsEnabled;
		AbstractMessageHandlerMetrics handlerMetrics = this.handlerMetrics;
		SampleFacade sample = null;
		if (countsEnabled && this.metricsCaptor != null) { // false
			sample = this.metricsCaptor.start();
		}
  
		try {
			if (this.shouldTrack) { // false
				message = MessageHistory.write(message, this, getMessageBuilderFactory());
			}
   
			if (countsEnabled) { // false
				start = handlerMetrics.beforeHandle();
				handleMessageInternal(message);
				if (sample != null) {
					sample.stop(sendTimer());
				}
				handlerMetrics.afterHandle(start, true);
			} else {
                           // *********************************************
                           // 5. 委托给子类处理
                           // AbstractMessageChannelBinder$SendingHandler
                           // *********************************************
				handleMessageInternal(message);
			}
		} catch (Exception e) {
			if (sample != null) {
				sample.stop(buildSendTimer(false, e.getClass().getSimpleName()));
			}
			if (countsEnabled) {
				handlerMetrics.afterHandle(start, false);
			}
			throw IntegrationUtils.wrapInHandlingExceptionIfNecessary(message,
					() -> "error occurred in message handler [" + this + "]", e);
		}
    } //end 
}
```
### (5).RocketMQMessageChannelBinder$SendingHandler
```
public abstract class AbstractMessageChannelBinder<C extends ConsumerProperties, P extends ProducerProperties, PP extends ProvisioningProvider<C, P>>
		extends AbstractBinder<MessageChannel, C, P>
		implements PollableConsumerBinder<MessageHandler, C>, ApplicationEventPublisherAware {


    private final class SendingHandler extends AbstractMessageHandler implements Lifecycle {
        
        protected void handleMessageInternal(Message<?> message) throws Exception {
            Message<?> messageToSend = (this.useNativeEncoding) ? message : serializeAndEmbedHeadersIfApplicable(message);
            // **************************************************************
            // **************************************************************
            // 6. 委托给:
            // com.alibaba.cloud.stream.binder.rocketmq.integration.RocketMQMessageHandler
            // 处理消息
            // **************************************************************
            // **************************************************************
            this.delegate.handleMessage(messageToSend);
        } //end handleMessageInternal
    } //end SendingHandler
    
    private Message<?> serializeAndEmbedHeadersIfApplicable(Message<?> message) throws Exception {
        // 拷贝message到MessageValues
        MessageValues transformed = serializePayloadIfNecessary(message);
        Object payload;
        if (this.embedHeaders) { // false
            Object contentType = transformed.get(MessageHeaders.CONTENT_TYPE);
            if (contentType != null) {
                transformed.put(MessageHeaders.CONTENT_TYPE, contentType.toString());
            }
            
            Object originalContentType = transformed.get(BinderHeaders.BINDER_ORIGINAL_CONTENT_TYPE);
            if (originalContentType != null) {
                transformed.put(BinderHeaders.BINDER_ORIGINAL_CONTENT_TYPE, originalContentType.toString());
            }
            
            payload = EmbeddedHeaderUtils.embedHeaders(transformed, this.embeddedHeaders);
        } else {
            payload = transformed.getPayload();
        }
        // 拷贝生产一个新的Message
        return getMessageBuilderFactory().withPayload(payload).copyHeaders(transformed.getHeaders()).build();
    }// end serializeAndEmbedHeadersIfApplicable
                  
}
```
### (6).RocketMQMessageHandler
```
public class RocketMQMessageHandler 
       extends AbstractMessageHandler 
       implements Lifecycle {
    
    // 处理消息
    protected void handleMessageInternal(org.springframework.messaging.Message<?> message) {
        try {
			
             Map<String, String> jsonHeaders = headerMapper.fromHeaders(message.getHeaders());
            message = org.springframework.messaging.support.MessageBuilder.fromMessage(message).copyHeaders(jsonHeaders).build();

            final StringBuilder topicWithTags = new StringBuilder(destination);
            String tags = Optional.ofNullable(message.getHeaders().get(RocketMQHeaders.TAGS)).orElse("").toString();
            if (!StringUtils.isEmpty(tags)) {
                topicWithTags.append(":").append(tags);
            }

            SendResult sendRes = null;
            // 是否为事务消息
            if (transactional) {
                    sendRes = rocketMQTemplate.sendMessageInTransaction(groupName,topicWithTags.toString(), message, message.getHeaders().get(RocketMQBinderConstants.ROCKET_TRANSACTIONAL_ARG));
                    log.debug("transactional send to topic " + topicWithTags + " " + sendRes);
            } else {
                    int delayLevel = 0;
                    try {
                           // 延迟级别
                            Object delayLevelObj = message.getHeaders().getOrDefault(MessageConst.PROPERTY_DELAY_TIME_LEVEL, 0);
                    	if (delayLevelObj instanceof Number) {
                    		delayLevel = ((Number) delayLevelObj).intValue();
                    	}
                    	else if (delayLevelObj instanceof String) {
                    		delayLevel = Integer.parseInt((String) delayLevelObj);
                    	}
                    } catch (Exception e) {
			}
   
                   // 
			boolean needSelectQueue = message.getHeaders().containsKey(BinderHeaders.PARTITION_HEADER);
			if (sync) {
                    	if (needSelectQueue) {
                    		sendRes = rocketMQTemplate.syncSendOrderly(
                    				topicWithTags.toString(), message, "",
                    				rocketMQTemplate.getProducer().getSendMsgTimeout());
                    	}
                    	else {
                    		sendRes = rocketMQTemplate.syncSend(topicWithTags.toString(),
                    				message,
                    				rocketMQTemplate.getProducer().getSendMsgTimeout(),
                    				delayLevel);
                    	}
                    	log.debug("sync send to topic " + topicWithTags + " " + sendRes);
			} else {
                            Message<?> finalMessage = message;
                            SendCallback sendCallback = new SendCallback() {
						@Override
						public void onSuccess(SendResult sendResult) {
							log.debug("async send to topic " + topicWithTags + " "
									+ sendResult);
						}

						@Override
						public void onException(Throwable e) {
							log.error("RocketMQ Message hasn't been sent. Caused by "
									+ e.getMessage());
							if (getSendFailureChannel() != null) {
								getSendFailureChannel().send(
										RocketMQMessageHandler.this.errorMessageStrategy
												.buildErrorMessage(new MessagingException(
														finalMessage, e), null));
							}
						}
				}; //end sendCallback
     
				if (needSelectQueue) {
						rocketMQTemplate.asyncSendOrderly(topicWithTags.toString(),
								message, "", sendCallback,
								rocketMQTemplate.getProducer().getSendMsgTimeout());
				} else {
						rocketMQTemplate.asyncSend(topicWithTags.toString(), message,
								sendCallback);
				}
			}// end else
            }//end else
   
            // 如果没有发送成功
            if (sendRes != null && !sendRes.getSendStatus().equals(SendStatus.SEND_OK)) {
                if (getSendFailureChannel() != null) {
                    this.getSendFailureChannel().send(message);
                } else {
                    throw new MessagingException(message, new MQClientException("message hasn't been sent", null));
                } // end else
            } //end if
        } catch (Exception e) {
            log.error("RocketMQ Message hasn't been sent. Caused by " + e.getMessage());
            if (getSendFailureChannel() != null) {
                    getSendFailureChannel().send(this.errorMessageStrategy.buildErrorMessage(new MessagingException(message, e), null));
            } else {
                    throw new MessagingException(message, e);
            }
        }// end catch

	}//end handleMessageInternal
}
```
### (7).总结