---
layout: post
title: 'RocketMQ源码之DefaultMQProducer介绍(一)' 
date: 2023-01-23
author: 李新
tags:  RocketMQ
---

### (1). Producer 案例
```
package help.lixin.example;

import java.util.List;

import org.apache.rocketmq.client.producer.DefaultMQProducer;
import org.apache.rocketmq.client.producer.MessageQueueSelector;
import org.apache.rocketmq.client.producer.SendResult;
import org.apache.rocketmq.common.message.Message;
import org.apache.rocketmq.common.message.MessageQueue;
import org.apache.rocketmq.remoting.common.RemotingHelper;

public class SyncProducer {
    public static void main(String[] args) throws Exception {
        DefaultMQProducer producer = new DefaultMQProducer("ProducerGroup");
        producer.setNamesrvAddr("127.0.0.1:9876");
        // 失败重试次数3
        producer.setRetryTimesWhenSendFailed(3);
        producer.start();

        for (int i = 0; i < 100; i++) {
            Message msg = new Message("TopicTest", "TagE", "id-" + String.valueOf(i),
                    (i + " - TagE Hello,World!!!").getBytes(RemotingHelper.DEFAULT_CHARSET));
            SendResult result = producer.send(msg);
            System.out.printf("%s%n", result);
        }
        producer.shutdown();
    }
}
```
### (2). DefaultMQProducer
```
public class DefaultMQProducer 
       extends ClientConfig 
       implements MQProducer {
 // DefaultMQProducerImpl  
 protected final transient DefaultMQProducerImpl defaultMQProducerImpl;           

 public DefaultMQProducer(final String producerGroup) {
    this(producerGroup, null);
 }
 
 public DefaultMQProducer(final String producerGroup, RPCHook rpcHook) {
    this.producerGroup = producerGroup;
    defaultMQProducerImpl = new DefaultMQProducerImpl(this, rpcHook);
 }
 
 // 启动
 public void start() throws MQClientException {
    this.defaultMQProducerImpl.start();
 }  
 
 // 同步发送
 public SendResult send(Message msg) 
        throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
    return this.defaultMQProducerImpl.send(msg);
 }
}
```
### (3). DefaultMQProducerImpl
```
public class DefaultMQProducerImpl 
       implements MQProducerInner {
  // 给用户一个配置
  private final DefaultMQProducer defaultMQProducer;
  // 设置调用前后处理
  private final RPCHook rpcHook;
  
  public DefaultMQProducerImpl(final DefaultMQProducer defaultMQProducer, RPCHook rpcHook) {
     this.defaultMQProducer = defaultMQProducer;
     this.rpcHook = rpcHook;
  }

 public void start() throws MQClientException {
        this.start(true);
 }

 public void start(final boolean startFactory) throws MQClientException {
    switch (this.serviceState) {
        case CREATE_JUST:
            this.serviceState = ServiceState.START_FAILED;

            this.checkConfig();
            // 配置clientId
            if (!this.defaultMQProducer.getProducerGroup().equals(MixAll.CLIENT_INNER_PRODUCER_GROUP)) {
                this.defaultMQProducer.changeInstanceNameToPID();
            }
            
            // 创建MQClient实例(包含:Netty请求等)
            this.mQClientFactory = MQClientManager.getInstance().getAndCreateMQClientInstance(this.defaultMQProducer, rpcHook);

            boolean registerOK = mQClientFactory.registerProducer(this.defaultMQProducer.getProducerGroup(), this);
            if (!registerOK) {
                this.serviceState = ServiceState.CREATE_JUST;
                throw new MQClientException("The producer group[" + this.defaultMQProducer.getProducerGroup()
                        + "] has been created before, specify another name please." + FAQUrl.suggestTodo(FAQUrl.GROUP_NAME_DUPLICATE_URL),
                        null);
            }

            this.topicPublishInfoTable.put(this.defaultMQProducer.getCreateTopicKey(), new TopicPublishInfo());

            if (startFactory) {
                // 调用:MQClientInstance.start()
                // Netty Start
                // 定时任务获取Topic信息
                mQClientFactory.start();
            }

            log.info("the producer [{}] start OK. sendMessageWithVIPChannel={}", this.defaultMQProducer.getProducerGroup(),
                this.defaultMQProducer.isSendMessageWithVIPChannel());
                this.serviceState = ServiceState.RUNNING;
            break;
        case RUNNING:
        case START_FAILED:
        case SHUTDOWN_ALREADY:
            throw new MQClientException("The producer service state not OK, maybe started once, "
                    + this.serviceState
                    + FAQUrl.suggestTodo(FAQUrl.CLIENT_SERVICE_NOT_OK),
                    null);
        default:
            break;
    }
    
    // 发送heartbeat
    this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();
  }
  
  public SendResult send(Message msg) 
        throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
        return send(msg, this.defaultMQProducer.getSendMsgTimeout());
  }
  
  public SendResult send(Message msg,long timeout) 
        throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
        return this.sendDefaultImpl(msg, CommunicationMode.SYNC, null, timeout);
  }
  
  // 发送消息的真正实现
  private SendResult sendDefaultImpl(
        Message msg,
        final CommunicationMode communicationMode,
        final SendCallback sendCallback,
        final long timeout
    ) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
		this.makeSureStateOK();
		Validators.checkMessage(msg, this.defaultMQProducer);

		final long invokeID = random.nextLong();
		long beginTimestampFirst = System.currentTimeMillis();
		long beginTimestampPrev = beginTimestampFirst;
		long endTimestamp = beginTimestampFirst;
           // 调用NameServer获取Topic对应的信息
		TopicPublishInfo topicPublishInfo = 
               this.tryToFindTopicPublishInfo(msg.getTopic());
		if (topicPublishInfo != null && topicPublishInfo.ok()) {
		    boolean callTimeout = false;
		    MessageQueue mq = null;
		    Exception exception = null;
		    SendResult sendResult = null;
               //  获得重试次数统计()
              //  同步模式下:timesTotal = 1 + retryTimesWhenSendFailed
              //  异步模式下:timesTotal = 1 
		    int timesTotal = 
                 communicationMode == CommunicationMode.SYNC 
                             ? 1 + this.defaultMQProducer.getRetryTimesWhenSendFailed() 
                             : 1;
		    int times = 0;
		    String[] brokersSent = new String[timesTotal];
		    for (; times < timesTotal; times++) {
			String lastBrokerName = null == mq ? null : mq.getBrokerName();
                   // 根据Topic获得一个Queue
			MessageQueue mqSelected = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName);
			if (mqSelected != null) {
			    mq = mqSelected;
			    brokersSent[times] = mq.getBrokerName();
			    try {
                          // 当前时间 
				beginTimestampPrev = System.currentTimeMillis();
                          // 当前时间 - 任务开发第一次的时间
				long costTime = beginTimestampPrev - beginTimestampFirst;
				if (timeout < costTime) {
                              // 超时任务取消掉
				    callTimeout = true;
				    break;
				}
                          
                          // 发送
                          // tryToCompressMessage(msg),如果消息 > 4096,则进行压缩
				sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, timeout - costTime);
				endTimestamp = System.currentTimeMillis();
				this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);
				switch (communicationMode) {
				    case ASYNC:
					return null;
				    case ONEWAY:
					return null;
				    case SYNC:
					if (sendResult.getSendStatus() != SendStatus.SEND_OK) {
					    if (this.defaultMQProducer.isRetryAnotherBrokerWhenNotStoreOK()) {
						continue;
					    }
					}
                                  //  return 会中止被调函数的调用.返回到调用函数端
					return sendResult;
				    default:
					break;
				}
			    } catch (RemotingException e) {
				endTimestamp = System.currentTimeMillis();
				this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, true);
				log.warn(String.format("sendKernelImpl exception, resend at once, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mq), e);
				log.warn(msg.toString());
				exception = e;
				continue;
			    } catch (MQClientException e) {
				endTimestamp = System.currentTimeMillis();
				this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, true);
				log.warn(String.format("sendKernelImpl exception, resend at once, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mq), e);
				log.warn(msg.toString());
				exception = e;
				continue;
			    } catch (MQBrokerException e) {
				endTimestamp = System.currentTimeMillis();
				this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, true);
				log.warn(String.format("sendKernelImpl exception, resend at once, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mq), e);
				log.warn(msg.toString());
				exception = e;
				switch (e.getResponseCode()) {
				    case ResponseCode.TOPIC_NOT_EXIST:
				    case ResponseCode.SERVICE_NOT_AVAILABLE:
				    case ResponseCode.SYSTEM_ERROR:
				    case ResponseCode.NO_PERMISSION:
				    case ResponseCode.NO_BUYER_ID:
				    case ResponseCode.NOT_IN_CURRENT_UNIT:
					continue;
				    default:
					if (sendResult != null) {
					    return sendResult;
					}

					throw e;
				}
			    } catch (InterruptedException e) {
				endTimestamp = System.currentTimeMillis();
				this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);
				log.warn(String.format("sendKernelImpl exception, throw exception, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mq), e);
				log.warn(msg.toString());

				log.warn("sendKernelImpl exception", e);
				log.warn(msg.toString());
				throw e;
			    }
			} else {
			    break;
			}
		    }

		    if (sendResult != null) {
			return sendResult;
		    }

		    String info = String.format("Send [%d] times, still failed, cost [%d]ms, Topic: %s, BrokersSent: %s",
			times,
			System.currentTimeMillis() - beginTimestampFirst,
			msg.getTopic(),
			Arrays.toString(brokersSent));

		    info += FAQUrl.suggestTodo(FAQUrl.SEND_MSG_FAILED);

		    MQClientException mqClientException = new MQClientException(info, exception);
		    if (callTimeout) {
			throw new RemotingTooMuchRequestException("sendDefaultImpl call timeout");
		    }

		    if (exception instanceof MQBrokerException) {
			mqClientException.setResponseCode(((MQBrokerException) exception).getResponseCode());
		    } else if (exception instanceof RemotingConnectException) {
			mqClientException.setResponseCode(ClientErrorCode.CONNECT_BROKER_EXCEPTION);
		    } else if (exception instanceof RemotingTimeoutException) {
			mqClientException.setResponseCode(ClientErrorCode.ACCESS_BROKER_TIMEOUT);
		    } else if (exception instanceof MQClientException) {
			mqClientException.setResponseCode(ClientErrorCode.BROKER_NOT_EXIST_EXCEPTION);
		    }

		    throw mqClientException;
		}

		List<String> nsList = this.getmQClientFactory().getMQClientAPIImpl().getNameServerAddressList();
		if (null == nsList || nsList.isEmpty()) {
		    throw new MQClientException(
			"No name server address, please set it." + FAQUrl.suggestTodo(FAQUrl.NAME_SERVER_ADDR_NOT_EXIST_URL), null).setResponseCode(ClientErrorCode.NO_NAME_SERVER_EXCEPTION);
		}

		throw new MQClientException("No route info of this topic, " + msg.getTopic() + FAQUrl.suggestTodo(FAQUrl.NO_TOPIC_ROUTE_INFO),
		    null).setResponseCode(ClientErrorCode.NOT_FOUND_TOPIC_EXCEPTION);
  }
  
  
  private SendResult sendKernelImpl(final Message msg,
	     final MessageQueue mq,
	     final CommunicationMode communicationMode,
	     final SendCallback sendCallback,
        final TopicPublishInfo topicPublishInfo,
	     final long timeout) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
		long beginStartTime = System.currentTimeMillis();
           // 
		String brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());
		if (null == brokerAddr) {
		    tryToFindTopicPublishInfo(mq.getTopic());
		    brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());
		}

		SendMessageContext context = null;
		if (brokerAddr != null) {
		    brokerAddr = MixAll.brokerVIPChannel(this.defaultMQProducer.isSendMessageWithVIPChannel(), brokerAddr);

		    byte[] prevBody = msg.getBody();
		    try {
			//for MessageBatch,ID has been set in the generating process
			if (!(msg instanceof MessageBatch)) {
			    MessageClientIDSetter.setUniqID(msg);
			}

			int sysFlag = 0;
			boolean msgBodyCompressed = false;
			if (this.tryToCompressMessage(msg)) {
			    sysFlag |= MessageSysFlag.COMPRESSED_FLAG;
			    msgBodyCompressed = true;
			}

			final String tranMsg = msg.getProperty(MessageConst.PROPERTY_TRANSACTION_PREPARED);
			if (tranMsg != null && Boolean.parseBoolean(tranMsg)) {
			    sysFlag |= MessageSysFlag.TRANSACTION_PREPARED_TYPE;
			}

			if (hasCheckForbiddenHook()) {
			    CheckForbiddenContext checkForbiddenContext = new CheckForbiddenContext();
			    checkForbiddenContext.setNameSrvAddr(this.defaultMQProducer.getNamesrvAddr());
			    checkForbiddenContext.setGroup(this.defaultMQProducer.getProducerGroup());
			    checkForbiddenContext.setCommunicationMode(communicationMode);
			    checkForbiddenContext.setBrokerAddr(brokerAddr);
			    checkForbiddenContext.setMessage(msg);
			    checkForbiddenContext.setMq(mq);
			    checkForbiddenContext.setUnitMode(this.isUnitMode());
			    this.executeCheckForbiddenHook(checkForbiddenContext);
			}

			if (this.hasSendMessageHook()) {
			    context = new SendMessageContext();
			    context.setProducer(this);
			    context.setProducerGroup(this.defaultMQProducer.getProducerGroup());
			    context.setCommunicationMode(communicationMode);
			    context.setBornHost(this.defaultMQProducer.getClientIP());
			    context.setBrokerAddr(brokerAddr);
			    context.setMessage(msg);
			    context.setMq(mq);
			    String isTrans = msg.getProperty(MessageConst.PROPERTY_TRANSACTION_PREPARED);
			    if (isTrans != null && isTrans.equals("true")) {
				context.setMsgType(MessageType.Trans_Msg_Half);
			    }

			    if (msg.getProperty("__STARTDELIVERTIME") != null || msg.getProperty(MessageConst.PROPERTY_DELAY_TIME_LEVEL) != null) {
				context.setMsgType(MessageType.Delay_Msg);
			    }
			    this.executeSendMessageHookBefore(context);
			}

			SendMessageRequestHeader requestHeader = new SendMessageRequestHeader();
                   // ProducerGroup
			requestHeader.setProducerGroup(this.defaultMQProducer.getProducerGroup());
                   // TopicTest
			requestHeader.setTopic(msg.getTopic());
                   // AUTO_CREATE_TOPIC_KEY
			requestHeader.setDefaultTopic(this.defaultMQProducer.getCreateTopicKey());
                   // 4
			requestHeader.setDefaultTopicQueueNums(this.defaultMQProducer.getDefaultTopicQueueNums());
                   // 5 (队列ID)
			requestHeader.setQueueId(mq.getQueueId());
                   // 0
			requestHeader.setSysFlag(sysFlag);
                   // 1570761529728
			requestHeader.setBornTimestamp(System.currentTimeMillis());
                   // 0
			requestHeader.setFlag(msg.getFlag());
                   // msg.getProperties() = 
                   // { 
                   //    KEYS=id-0, 
                   //    UNIQ_KEY=0A0006525E6418B4AAC235C5D7990000, 
                   //    WAIT=true,
                   //    TAGS=TagE
                   // }
			requestHeader.setProperties(MessageDecoder.messageProperties2String(msg.getProperties()));
			requestHeader.setReconsumeTimes(0);
                   // false
			requestHeader.setUnitMode(this.isUnitMode());
                   // false
			requestHeader.setBatch(msg instanceof MessageBatch);
                   // 判断队列是否以: %RETRY% 开头
			if (requestHeader.getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
			    String reconsumeTimes = MessageAccessor.getReconsumeTime(msg);
			    if (reconsumeTimes != null) {
				requestHeader.setReconsumeTimes(Integer.valueOf(reconsumeTimes));
				MessageAccessor.clearProperty(msg, MessageConst.PROPERTY_RECONSUME_TIME);
			    }

			    String maxReconsumeTimes = MessageAccessor.getMaxReconsumeTimes(msg);
			    if (maxReconsumeTimes != null) {
				requestHeader.setMaxReconsumeTimes(Integer.valueOf(maxReconsumeTimes));
				MessageAccessor.clearProperty(msg, MessageConst.PROPERTY_MAX_RECONSUME_TIMES);
			    }
			}
                  
                  
			SendResult sendResult = null;
			switch (communicationMode) {  
			    case ASYNC:                  // 异步处理
				Message tmpMessage = msg;
				if (msgBodyCompressed) {
				    //If msg body was compressed, msgbody should be reset using prevBody.
				    //Clone new message using commpressed message body and recover origin massage.
				    //Fix bug:https://github.com/apache/rocketmq-externals/issues/66
				    tmpMessage = MessageAccessor.cloneMessage(msg);
				    msg.setBody(prevBody);
				}
				long costTimeAsync = System.currentTimeMillis() - beginStartTime;
				if (timeout < costTimeAsync) {
				    throw new RemotingTooMuchRequestException("sendKernelImpl call timeout");
				}
				sendResult = this.mQClientFactory.getMQClientAPIImpl().sendMessage(
				    brokerAddr,
				    mq.getBrokerName(),
				    tmpMessage,
				    requestHeader,
				    timeout - costTimeAsync,
				    communicationMode,
				    sendCallback,    // 异步回调处理
				    topicPublishInfo,
				    this.mQClientFactory,
				    this.defaultMQProducer.getRetryTimesWhenSendAsyncFailed(),
				    context,
				    this);
				break;
			    case ONEWAY:                   // 同步处理
			    case SYNC:                     // 同步处理
				long costTimeSync = System.currentTimeMillis() - beginStartTime;
				if (timeout < costTimeSync) {
				    throw new RemotingTooMuchRequestException("sendKernelImpl call timeout");
				}
                           // 重点:真实的发送消息
				sendResult = this.mQClientFactory.getMQClientAPIImpl().sendMessage(
				    brokerAddr,
				    mq.getBrokerName(),
				    msg,
				    requestHeader,
				    timeout - costTimeSync,
				    communicationMode,
				    context,
				    this);
				break;
			    default:
				assert false;
				break;
			}

			if (this.hasSendMessageHook()) {
			    context.setSendResult(sendResult);
			    this.executeSendMessageHookAfter(context);
			}

			return sendResult;
		    } catch (RemotingException e) {
			if (this.hasSendMessageHook()) {
			    context.setException(e);
			    this.executeSendMessageHookAfter(context);
			}
			throw e;
		    } catch (MQBrokerException e) {
			if (this.hasSendMessageHook()) {
			    context.setException(e);
			    this.executeSendMessageHookAfter(context);
			}
			throw e;
		    } catch (InterruptedException e) {
			if (this.hasSendMessageHook()) {
			    context.setException(e);
			    this.executeSendMessageHookAfter(context);
			}
			throw e;
		    } finally {
			msg.setBody(prevBody);
		    }
		}
		throw new MQClientException("The broker[" + mq.getBrokerName() + "] not exist", null);
  }
  
}
```
### (4). MQClientAPIImpl
```
public class MQClientAPIImpl {
    public SendResult sendMessage(
        final String addr,
        final String brokerName,
        final Message msg,
        final SendMessageRequestHeader requestHeader,
        final long timeoutMillis,
        final CommunicationMode communicationMode,
        final SendMessageContext context,
        final DefaultMQProducerImpl producer
    ) throws RemotingException, MQBrokerException, InterruptedException {
        return sendMessage(addr, brokerName, msg, requestHeader, timeoutMillis, communicationMode, null, null, null, 0, context, producer);
    }
    
    public SendResult sendMessage(
        final String addr,
        final String brokerName,
        final Message msg,
        final SendMessageRequestHeader requestHeader,
        final long timeoutMillis,
        final CommunicationMode communicationMode,
        final SendCallback sendCallback,
        final TopicPublishInfo topicPublishInfo,
        final MQClientInstance instance,
        final int retryTimesWhenSendFailed,
        final SendMessageContext context,
        final DefaultMQProducerImpl producer
    ) throws RemotingException, MQBrokerException, InterruptedException {
        long beginStartTime = System.currentTimeMillis();
        RemotingCommand request = null;
        if (sendSmartMsg || msg instanceof MessageBatch) {
            SendMessageRequestHeaderV2 requestHeaderV2 = SendMessageRequestHeaderV2.createSendMessageRequestHeaderV2(requestHeader);
            request = RemotingCommand.createRequestCommand(msg instanceof MessageBatch ? RequestCode.SEND_BATCH_MESSAGE : RequestCode.SEND_MESSAGE_V2, requestHeaderV2);
        } else {
            request = RemotingCommand.createRequestCommand(RequestCode.SEND_MESSAGE, requestHeader);
        }

        request.setBody(msg.getBody());

        switch (communicationMode) {
            case ONEWAY:   // 单次发送.不论成功或失败
                this.remotingClient.invokeOneway(addr, request, timeoutMillis);
                return null;
            case ASYNC:   // 异步发送,通过回调处理
                final AtomicInteger times = new AtomicInteger();
                long costTimeAsync = System.currentTimeMillis() - beginStartTime;
                if (timeoutMillis < costTimeAsync) {
                    throw new RemotingTooMuchRequestException("sendMessage call timeout");
                }
                this.sendMessageAsync(addr, brokerName, msg, timeoutMillis - costTimeAsync, request, sendCallback, topicPublishInfo, instance,
                    retryTimesWhenSendFailed, times, context, producer);
                return null;
            case SYNC:  // 同步发送,等待结果返回
                long costTimeSync = System.currentTimeMillis() - beginStartTime;
                if (timeoutMillis < costTimeSync) {
                    throw new RemotingTooMuchRequestException("sendMessage call timeout");
                }
                return this.sendMessageSync(addr, brokerName, msg, timeoutMillis - costTimeSync, request);
            default:
                assert false;
                break;
        }

        return null;
    }
    
    private SendResult sendMessageSync(
        final String addr,
        final String brokerName,
        final Message msg,
        final long timeoutMillis,
        final RemotingCommand request
    ) throws RemotingException, MQBrokerException, InterruptedException {
        // 调用Brokder服务器
        RemotingCommand response = this.remotingClient.invokeSync(addr, request, timeoutMillis);
        assert response != null;
        // 对返回结果进行处理
        return this.processSendResponse(brokerName, msg, response);
    }
    
    private SendResult processSendResponse(
        final String brokerName,
        final Message msg,
        final RemotingCommand response
    ) throws MQBrokerException, RemotingCommandException {
        switch (response.getCode()) {
            case ResponseCode.FLUSH_DISK_TIMEOUT:
            case ResponseCode.FLUSH_SLAVE_TIMEOUT:
            case ResponseCode.SLAVE_NOT_AVAILABLE: {
            }
            case ResponseCode.SUCCESS: {
                SendStatus sendStatus = SendStatus.SEND_OK;
                switch (response.getCode()) {
                    case ResponseCode.FLUSH_DISK_TIMEOUT:
                        sendStatus = SendStatus.FLUSH_DISK_TIMEOUT;
                        break;
                    case ResponseCode.FLUSH_SLAVE_TIMEOUT:
                        sendStatus = SendStatus.FLUSH_SLAVE_TIMEOUT;
                        break;
                    case ResponseCode.SLAVE_NOT_AVAILABLE:
                        sendStatus = SendStatus.SLAVE_NOT_AVAILABLE;
                        break;
                    case ResponseCode.SUCCESS:
                        sendStatus = SendStatus.SEND_OK;
                        break;
                    default:
                        assert false;
                        break;
                }
                
                // 对response进行解码
                SendMessageResponseHeader responseHeader =
                    (SendMessageResponseHeader) response.decodeCommandCustomHeader(SendMessageResponseHeader.class);
                // 创建MessageQueue
                MessageQueue messageQueue = new MessageQueue(msg.getTopic(), brokerName, responseHeader.getQueueId());
                
                // 获得消息的唯一ID
                String uniqMsgId = MessageClientIDSetter.getUniqID(msg);
                if (msg instanceof MessageBatch) {
                    StringBuilder sb = new StringBuilder();
                    for (Message message : (MessageBatch) msg) {
                        sb.append(sb.length() == 0 ? "" : ",").append(MessageClientIDSetter.getUniqID(message));
                    }
                    uniqMsgId = sb.toString();
                }
                // 创建发送SendResult对象
                SendResult sendResult = new SendResult(sendStatus,
                    uniqMsgId,
                    responseHeader.getMsgId(), messageQueue, responseHeader.getQueueOffset());
                // null
                sendResult.setTransactionId(responseHeader.getTransactionId());
                // DefaultRegion
                String regionId = response.getExtFields().get(MessageConst.PROPERTY_MSG_REGION);
                // "true"
                String traceOn = response.getExtFields().get(MessageConst.PROPERTY_TRACE_SWITCH);
                if (regionId == null || regionId.isEmpty()) {
                    regionId = MixAll.DEFAULT_TRACE_REGION_ID;
                }
                if (traceOn != null && traceOn.equals("false")) {
                    sendResult.setTraceOn(false);
                } else {
                    sendResult.setTraceOn(true);
                }
                sendResult.setRegionId(regionId);
                return sendResult;
            }
            default:
                break;
        }

        throw new MQBrokerException(response.getCode(), response.getRemark());
    } 
}
```
