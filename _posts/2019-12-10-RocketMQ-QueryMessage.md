---
layout: post
title: 'RocketMQ源码之根据主题或Key查询消息(二)' 
date: 2023-01-23
author: 李新
tags:  RocketMQ
---

### (1). 概述
前面对DefaultMQPullConsumer进行了简单的介绍,这一篇,直接绕过DefaultMQPullConsumer,而是直面底层的RPC调用,并模拟调用,目的在于,能更清晰的了解RocketMQ的功能,可以自由组合这些API. 

### (2). 根据Topic和Key查询消息
```
package help.lixin.example;

import java.nio.ByteBuffer;
import java.util.Calendar;
import java.util.LinkedList;
import java.util.List;
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

import org.apache.rocketmq.client.ClientConfig;
import org.apache.rocketmq.client.QueryResult;
import org.apache.rocketmq.common.MixAll;
import org.apache.rocketmq.common.message.MessageDecoder;
import org.apache.rocketmq.common.message.MessageExt;
import org.apache.rocketmq.common.protocol.RequestCode;
import org.apache.rocketmq.common.protocol.ResponseCode;
import org.apache.rocketmq.common.protocol.header.QueryMessageRequestHeader;
import org.apache.rocketmq.common.protocol.header.QueryMessageResponseHeader;
import org.apache.rocketmq.remoting.InvokeCallback;
import org.apache.rocketmq.remoting.RemotingClient;
import org.apache.rocketmq.remoting.exception.RemotingCommandException;
import org.apache.rocketmq.remoting.netty.NettyClientConfig;
import org.apache.rocketmq.remoting.netty.NettyRemotingClient;
import org.apache.rocketmq.remoting.netty.ResponseFuture;
import org.apache.rocketmq.remoting.protocol.RemotingCommand;

public class QueryMessageRequestTest {
    public static void main(String[] args) throws Exception {
        // 设置查询起始时间
        Calendar calendar = Calendar.getInstance();
        calendar.add(Calendar.DAY_OF_MONTH, -10);

        // 创建请求
        QueryMessageRequestHeader requestHeader = new QueryMessageRequestHeader();
        requestHeader.setTopic("TopicTest");
        // 生产者在创建消息时的keys[ Message(String topic, String tags, String keys, byte[] body) ]
        requestHeader.setKey("id-1");
        requestHeader.setMaxNum(10);
        requestHeader.setBeginTimestamp(calendar.getTimeInMillis());
        requestHeader.setEndTimestamp(System.currentTimeMillis());

        // 创建命令
        RemotingCommand request = RemotingCommand.createRequestCommand(RequestCode.QUERY_MESSAGE, requestHeader);
        Boolean isUnqiueKey = Boolean.FALSE;
        request.addExtField(MixAll.UNIQUE_MSG_QUERY_FLAG, isUnqiueKey.toString());

        // Netty
        NettyClientConfig nettyClientConfig = new NettyClientConfig();
        RemotingClient remotingClient = new NettyRemotingClient(nettyClientConfig);
        remotingClient.start();
        // 与Border通信并不需要NameServer的参与
        // remotingClient.updateNameServerAddressList(Arrays.asList("127.0.0.1:9876"));

        ClientConfig clientConfig = new ClientConfig();
        // MQ Brokder服务器地址(从NameServer中获取到的)
        String brokderAddr = "10.0.6.82:10911";

        // 查询结果集
        final List<QueryResult> queryResultList = new LinkedList<QueryResult>();
        final ReadWriteLock lock = new ReentrantReadWriteLock(false);

        // 同步获取方式
        RemotingCommand response = remotingClient.invokeSync(brokderAddr, request, 1000 * 3);
        switch (response.getCode()) {
            case ResponseCode.SUCCESS: {
                QueryMessageResponseHeader responseHeader = null;
                try {
                    responseHeader = (QueryMessageResponseHeader) response
                            .decodeCommandCustomHeader(QueryMessageResponseHeader.class);
                } catch (RemotingCommandException e) {
                    return;
                }
                List<MessageExt> wrappers = MessageDecoder.decodes(ByteBuffer.wrap(response.getBody()), true);
                QueryResult qr = new QueryResult(responseHeader.getIndexLastUpdateTimestamp(), wrappers);
                try {
                    lock.writeLock().lock();
                    queryResultList.add(qr);
                } finally {
                    lock.writeLock().unlock();
                }
                break;
            }
            default:
                // TODO
                break;
        }

        // 异步处理方式
        remotingClient.invokeAsync(MixAll.brokerVIPChannel(clientConfig.isVipChannelEnabled(), brokderAddr), request,
                1000 * 3, new InvokeCallback() {
                    @Override
                    public void operationComplete(ResponseFuture responseFuture) {
                        try {
                            RemotingCommand response = responseFuture.getResponseCommand();
                            if (response != null) {
                                switch (response.getCode()) {
                                    case ResponseCode.SUCCESS: {
                                        QueryMessageResponseHeader responseHeader = null;
                                        try {
                                            responseHeader = (QueryMessageResponseHeader) response
                                                    .decodeCommandCustomHeader(QueryMessageResponseHeader.class);
                                        } catch (RemotingCommandException e) {
                                            return;
                                        }
                                        List<MessageExt> wrappers = MessageDecoder
                                                .decodes(ByteBuffer.wrap(response.getBody()), true);
                                        QueryResult qr = new QueryResult(responseHeader.getIndexLastUpdateTimestamp(),
                                                wrappers);
                                        try {
                                            lock.writeLock().lock();
                                            queryResultList.add(qr);
                                        } finally {
                                            lock.writeLock().unlock();
                                        }
                                        break;
                                    }
                                    default:
                                        // TODO
                                        break;
                                }
                            } else {
                                // TODO
                            }
                        } finally {
                            // TODO
                        }
                    }
                });
    }
}
```
### (3). 查询结果后数据结构图
!["查询结果数据结构图"](/assets/rocketmq/imgs/rocketmq-querymessage.png)