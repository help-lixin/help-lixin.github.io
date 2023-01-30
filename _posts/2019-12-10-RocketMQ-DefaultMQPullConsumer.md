---
layout: post
title: 'RocketMQ源码之DefaultMQPullConsumer介绍(一)' 
date: 2023-01-23
author: 李新
tags:  RocketMQ
---

### (1). DefaultMQPullConsumer 的继承关系
```
org.apache.rocketmq.client.MQAdmin
  org.apache.rocketmq.client.consumer.MQConsumer
    org.apache.rocketmq.client.consumer.MQPullConsumer
     org.apache.rocketmq.client.consumer.DefaultMQPullConsumer
```
### (2). 查看MQAdmin职责
```
# 创建Topic
void createTopic(final String key, 
                 final String newTopic, 
                 final int queueNum)
        throws MQClientException;

# 创建Topic
void createTopic(String key, 
                String newTopic, 
                int queueNum, 
                int topicSysFlag)
        throws MQClientException;

# 搜索offset
long searchOffset(final MessageQueue mq, 
            final long timestamp) throws MQClientException;


# 查询最大的offset
long maxOffset(final MessageQueue mq) 
                          throws MQClientException;

# 查询最小的offset
long minOffset(final MessageQueue mq) 
                          throws MQClientException;

# 查询最早的存储时间
long earliestMsgStoreTime(final MessageQueue mq) 
                         throws MQClientException;

# 根据offsetMsgId查看消息信息
MessageExt viewMessage(final String offsetMsgId) 
         throws RemotingException, MQBrokerException,
        InterruptedException, MQClientException;
        
# 根据topic和msgid获得消息
 MessageExt viewMessage(String topic,
                         String msgId) 
        throws RemotingException, MQBrokerException, InterruptedException, MQClientException;

# 查询消息(根据topic/key/maxNum/begin/end)
QueryResult queryMessage(final String topic, 
                         final String key, 
                         final int maxNum, 
                         final long begin,
                         final long end) 
         throws MQClientException, InterruptedException;                
```
### (3). 查看MQConsumer的职责
```
# 发送消息
void sendMessageBack(final MessageExt msg, 
                     final int delayLevel, 
                     final String brokerName)
        throws RemotingException, MQBrokerException, InterruptedException, MQClientException;
        
# 根据Topic获取消息
Set<MessageQueue> fetchSubscribeMessageQueues(final String topic) 
         throws MQClientException;                

```
### (4). 查看MQPullConsumer的职责
```
# 拉取消息
PullResult pull(final MessageQueue mq, 
                final String subExpression, 
                final long offset,
                final int maxNums) 
throws MQClientException, RemotingException, MQBrokerException,InterruptedException;
```
### (5). 查看DefaultMQPullConsumer的职责
```
# DefaultMQPullConsumer 将所有的请求委托给:DefaultMQPullConsumerImpl进行处理
public class DefaultMQPullConsumer extends 
           ClientConfig implements MQPullConsumer {
    protected final transient 
                   DefaultMQPullConsumerImpl defaultMQPullConsumerImpl;
}
```
### (6). 查看MQConsumerInner的职责
```
# 消费者组名称
String groupName();

# 消息模式(BROADCASTING/CLUSTERING)
# BROADCASTING:消费位置存储在Client(LocalFileOffsetStore)
# CLUSTERING:存储在远端(RemoteBrokerOffsetStore)
MessageModel messageModel();

# 消费者类型(PULL/PUSH)
ConsumeType consumeType();

# CONSUME_FROM_LAST_OFFSET : 默认策略,从该队列最尾开始消费,即跳过历史消息.
# CONSUME_FROM_FIRST_OFFSET : 从队列最开始开始消费,即历史消息(还储存在broker的)全部消费一遍
# CONSUME_FROM_TIMESTAMP : 从某个时间点开始消费,和setConsumeTimestamp()配合使用,
# 默认是半个小时以前
ConsumeFromWhere consumeFromWhere();

# 订阅了哪些主题
Set<SubscriptionData> subscriptions();

# 重新负载均衡下
void doRebalance();

# 持久化consumer的消费位置
void persistConsumerOffset();

# 更新topic订阅信息
void updateTopicSubscribeInfo(final String topic, final Set<MessageQueue> info);

# 是否订阅更新
boolean isSubscribeTopicNeedUpdate(final String topic);

boolean isUnitMode();

# 查看消费者运行状态
ConsumerRunningInfo consumerRunningInfo();
```
### (7). 以获取Topic路由信息为案例
```
public class MQClientAPIImpl {
   public TopicRouteData getTopicRouteInfoFromNameServer(final String topic, final long timeoutMillis,
        boolean allowTopicNotExist) throws MQClientException, InterruptedException, RemotingTimeoutException, RemotingSendRequestException, RemotingConnectException {
        GetRouteInfoRequestHeader requestHeader = new GetRouteInfoRequestHeader();
        requestHeader.setTopic(topic);

        RemotingCommand request = RemotingCommand.createRequestCommand(RequestCode.GET_ROUTEINTO_BY_TOPIC, requestHeader);

        RemotingCommand response = this.remotingClient.invokeSync(null, request, timeoutMillis);
        assert response != null;
        switch (response.getCode()) {
            case ResponseCode.TOPIC_NOT_EXIST: {
                if (allowTopicNotExist && !topic.equals(MixAll.AUTO_CREATE_TOPIC_KEY_TOPIC)) {
                    log.warn("get Topic [{}] RouteInfoFromNameServer is not exist value", topic);
                }

                break;
            }
            case ResponseCode.SUCCESS: {
                byte[] body = response.getBody();
                if (body != null) {
                    return TopicRouteData.decode(body, TopicRouteData.class);
                }
            }
            default:
                break;
        }

        throw new MQClientException(response.getCode(), response.getRemark());
    }
}
```
### (8). 总结
阿里的代码有一个习惯,那就是:基本上是面向接口编程,然后,接口的职责比较明细.  