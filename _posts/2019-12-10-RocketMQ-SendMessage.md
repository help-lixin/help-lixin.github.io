---
layout: post
title: 'RocketMQ源码之生产消息(二)' 
date: 2019-12-10
author: 李新
tags:  RocketMQ
---

### (1). 概述
前面对DefaultMQPullConsumer进行了简单的介绍,这一篇,直接绕过DefaultMQPullConsumer,而是直面底层的RPC调用,并模拟调用,目的在于,能更清晰的了解RocketMQ的功能,可以自由组合这些API. 

### (2). 生产消息
```
package help.lixin.example;

import java.util.HashMap;
import java.util.Map;
import java.util.Properties;

import org.apache.rocketmq.client.producer.SendResult;
import org.apache.rocketmq.client.producer.SendStatus;
import org.apache.rocketmq.common.MixAll;
import org.apache.rocketmq.common.message.Message;
import org.apache.rocketmq.common.message.MessageBatch;
import org.apache.rocketmq.common.message.MessageClientIDSetter;
import org.apache.rocketmq.common.message.MessageConst;
import org.apache.rocketmq.common.message.MessageDecoder;
import org.apache.rocketmq.common.message.MessageQueue;
import org.apache.rocketmq.common.protocol.RequestCode;
import org.apache.rocketmq.common.protocol.ResponseCode;
import org.apache.rocketmq.common.protocol.header.SendMessageRequestHeader;
import org.apache.rocketmq.common.protocol.header.SendMessageRequestHeaderV2;
import org.apache.rocketmq.common.protocol.header.SendMessageResponseHeader;
import org.apache.rocketmq.remoting.common.RemotingHelper;
import org.apache.rocketmq.remoting.netty.NettyClientConfig;
import org.apache.rocketmq.remoting.netty.NettyRemotingClient;
import org.apache.rocketmq.remoting.protocol.RemotingCommand;

public class SendMessageTest {
    public static void main(String[] args) throws Exception {
        String brokerAddr = "10.0.6.82:10911";
        String brokerName = "k-161019-04-PC";

        NettyClientConfig nettyClientConfig = new NettyClientConfig();
        NettyRemotingClient remotingClient = new NettyRemotingClient(nettyClientConfig);
        remotingClient.start();

        Message msg = new Message("TopicTest", "TagE", "id-" + String.valueOf(0),
                (0 + " - TagE Hello,World B!!!").getBytes(RemotingHelper.DEFAULT_CHARSET));
        MessageClientIDSetter.setUniqID(msg);
        
        Map<String, String> properties =msg.getProperties();

        SendMessageRequestHeader requestHeader = new SendMessageRequestHeader();
        requestHeader.setProducerGroup("ProducerGroup");
        requestHeader.setTopic(msg.getTopic());
        requestHeader.setDefaultTopic("AUTO_CREATE_TOPIC_KEY");
        requestHeader.setDefaultTopicQueueNums(4);
        requestHeader.setQueueId(5);
        requestHeader.setSysFlag(0);
        requestHeader.setBornTimestamp(System.currentTimeMillis());
        requestHeader.setFlag(0);
        requestHeader.setProperties(MessageDecoder.messageProperties2String(properties));
        requestHeader.setReconsumeTimes(0);
        requestHeader.setUnitMode(Boolean.FALSE);
        requestHeader.setBatch(Boolean.FALSE);

        SendResult sendResult = null;

        SendMessageRequestHeaderV2 requestHeaderV2 = SendMessageRequestHeaderV2
                .createSendMessageRequestHeaderV2(requestHeader);
        RemotingCommand request = RemotingCommand.createRequestCommand(RequestCode.SEND_MESSAGE_V2, requestHeaderV2);
        request.setBody(msg.getBody());

        RemotingCommand response = remotingClient.invokeSync(brokerAddr, request, 1000 * 30);

        if (response.getCode() == ResponseCode.SUCCESS) {
            SendStatus sendStatus = SendStatus.SEND_OK;

            SendMessageResponseHeader responseHeader = (SendMessageResponseHeader) response
                    .decodeCommandCustomHeader(SendMessageResponseHeader.class);

            MessageQueue messageQueue = new MessageQueue(msg.getTopic(), brokerName, responseHeader.getQueueId());
            String uniqMsgId = MessageClientIDSetter.getUniqID(msg);
            sendResult = new SendResult(sendStatus, uniqMsgId, responseHeader.getMsgId(), messageQueue,
                    responseHeader.getQueueOffset());
            sendResult.setTransactionId(responseHeader.getTransactionId());
            String regionId = response.getExtFields().get(MessageConst.PROPERTY_MSG_REGION);
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
        }
    }
}
```