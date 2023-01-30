---
layout: post
title: 'RocketMQ源码之根据主题获取路由信息(二)' 
date: 2019-12-10
author: 李新
tags:  RocketMQ
---

### (1). 概述
前面对DefaultMQPullConsumer进行了简单的介绍,这一篇,直接绕过DefaultMQPullConsumer,而是直面底层的RPC调用,并模拟调用,目的在于,能更清晰的了解RocketMQ的功能,可以自由组合这些API. 

### (2). 根据Topic获取路由信息
```
public class GetRouteInfoRequestTest {
    public static void main(String[] args) throws Exception {
        GetRouteInfoRequestHeader requestHeader = new GetRouteInfoRequestHeader();
        requestHeader.setTopic("TopicTest");
        RemotingCommand request = RemotingCommand.createRequestCommand(RequestCode.GET_ROUTEINTO_BY_TOPIC,
                requestHeader);
        
        NettyClientConfig nettyClientConfig = new NettyClientConfig();
        RemotingClient remotingClient = new NettyRemotingClient(nettyClientConfig);
        //  初始化netty bootstartp
        remotingClient.start();
        // 设置NameServer地址
        remotingClient.updateNameServerAddressList(Arrays.asList("127.0.0.1:9876"));
        RemotingCommand response = remotingClient.invokeSync(null, request, 1000 * 3);
        TopicRouteData topicRouteData = null;
        switch (response.getCode()) {
            case ResponseCode.TOPIC_NOT_EXIST: {
                // TODO
                break;
            }
            case ResponseCode.SUCCESS: {
                byte[] body = response.getBody();
                if (body != null) {
                    topicRouteData = TopicRouteData.decode(body, TopicRouteData.class);
                }
            }
            default:
                break;
        }
        System.out.println(topicRouteData);
    }
}
```
### (3). 查询结果后数据结构图
!["查询结果后数据结构图"](/assets/rocketmq/imgs/rocketmq-getrouteinfo.png)
