---
layout: post
title: 'RocketMQ源码之拉取消息(三)' 
date: 2023-01-23
author: 李新
tags:  RocketMQ
---

### (1). 概述
前面对DefaultMQPullConsumer进行了简单的介绍,这一篇,直接绕过DefaultMQPullConsumer,而是直面底层的RPC调用,并模拟调用,目的在于,能更清晰的了解RocketMQ的功能,可以自由组合这些API. 

### (2). 自定义DefaultMQPullConsumer.pullBlockIfNotFound拉取消息
```
public class PullMessageTest {
    public static void main(String[] args) throws Exception {
        // TODO 应该要从 NameServer 读取
        // brokder address
        String addr = "10.0.6.82:10911";
        String consumerGroup = "test";
        // TODO 应该要从NameServer读取
        String topic = "TopicTest";
        // TODO 应该要从NameServer读取
        int queueId = 1;
        long offset = 0;
        int maxNums = 10;
        int sysFlagInner = 6;
        long commitOffset = 0;
        long brokerSuspendMaxTimeMillis = 20000;
        String subExpression = "*";
        long subVersion = 0;
        String expressionType = "TAG";

        // 创建拉取请求
        PullMessageRequestHeader requestHeader = new PullMessageRequestHeader();
        requestHeader.setConsumerGroup(consumerGroup);
        requestHeader.setTopic(topic);
        requestHeader.setQueueId(queueId);
        requestHeader.setQueueOffset(offset);
        requestHeader.setMaxMsgNums(maxNums);
        requestHeader.setSysFlag(sysFlagInner);
        requestHeader.setCommitOffset(commitOffset);
        requestHeader.setSuspendTimeoutMillis(brokerSuspendMaxTimeMillis);
        requestHeader.setSubscription(subExpression);
        requestHeader.setSubVersion(subVersion);
        requestHeader.setExpressionType(expressionType);

        // 创建请求
        RemotingCommand request = RemotingCommand.createRequestCommand(RequestCode.PULL_MESSAGE, requestHeader);

        NettyClientConfig nettyClientConfig = new NettyClientConfig();
        RemotingClient remotingClient = new NettyRemotingClient(nettyClientConfig);
        remotingClient.start();
        // 从Broder拉取消息,不需要NameServer信息
        // remotingClient.updateNameServerAddressList(Arrays.asList("127.0.0.1:9876"));

        // 发起远程请求
        RemotingCommand response = remotingClient.invokeSync(addr, request, 1000 * 3);
        PullStatus pullStatus = PullStatus.NO_NEW_MSG;
        switch (response.getCode()) {
            case ResponseCode.SUCCESS:
                // 如果有消息
                pullStatus = PullStatus.FOUND;
                break;
            case ResponseCode.PULL_NOT_FOUND:
                pullStatus = PullStatus.NO_NEW_MSG;
                break;
            case ResponseCode.PULL_RETRY_IMMEDIATELY:
                pullStatus = PullStatus.NO_MATCHED_MSG;
                break;
            case ResponseCode.PULL_OFFSET_MOVED:
                pullStatus = PullStatus.OFFSET_ILLEGAL;
                break;
            default:
                throw new MQBrokerException(response.getCode(), response.getRemark());
        }
        // 解码
        PullMessageResponseHeader responseHeader = (PullMessageResponseHeader) response
                .decodeCommandCustomHeader(PullMessageResponseHeader.class);
        PullResultExt pullResult = new PullResultExt(pullStatus, responseHeader.getNextBeginOffset(),
                responseHeader.getMinOffset(), responseHeader.getMaxOffset(), null,
                responseHeader.getSuggestWhichBrokerId(), response.getBody());

        // 解码body
        if (PullStatus.FOUND == pullResult.getPullStatus()) {
            ByteBuffer byteBuffer = ByteBuffer.wrap(pullResult.getMessageBinary());
            List<MessageExt> msgList = MessageDecoder.decodes(byteBuffer);

            List<MessageExt> msgListFilterAgain = msgList;
            for (MessageExt msg : msgListFilterAgain) {
                // 事务支持
                String traFlag = msg.getProperty(MessageConst.PROPERTY_TRANSACTION_PREPARED);
                if (traFlag != null && Boolean.parseBoolean(traFlag)) {
                    msg.setTransactionId(msg.getProperty(MessageConst.PROPERTY_UNIQ_CLIENT_MESSAGE_ID_KEYIDX));
                }
                // 设置minOffset
                MessageAccessor.putProperty(msg, MessageConst.PROPERTY_MIN_OFFSET,
                        Long.toString(pullResult.getMinOffset()));
                // 设置maxOffset
                MessageAccessor.putProperty(msg, MessageConst.PROPERTY_MAX_OFFSET,
                        Long.toString(pullResult.getMaxOffset()));
            }
            pullResult.setMsgFoundList(msgListFilterAgain);
        }
        pullResult.setMessageBinary(null);
        System.out.println(pullResult);
    }
}
```