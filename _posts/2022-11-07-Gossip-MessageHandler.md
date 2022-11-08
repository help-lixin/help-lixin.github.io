---
layout: post
title: 'Gossip源码之MessageHandler(二)' 
date: 2022-11-07
author: 李新
tags:  Gossip
---


### (1). 概述
分析Gossip源码时,发现创建:GossipManager对象时,会通过工厂(MessageHandlerFactory)创建:MessageHandler,所以,这一小篇,主要剖析它. 

### (2). GossipManagerBuilder构建GossipManager
```
GossipSettings s = new GossipSettings();
s.setWindowSize(1000);
s.setGossipInterval(100);

// GossipManager通过Builder模式创建
GossipManager gossipService = GossipManagerBuilder.newBuilder()
        .cluster("mycluster")
		// 本机成员信息,以及唯一id
		// udp://localhost:10000 0
		.uri(URI.create("udp://localhost:10000")).id("0")
		// 其它成员信息,以及,其它成员的唯一id
		// udp://localhost:10001 1
		.gossipMembers(Collections.singletonList(new RemoteMember("mycluster", URI.create("udp://localhost:10001"), "1")))
		.gossipSettings(s)
		.build();
```
### (3). GossipManagerBuilder.build
```
public GossipManager build() {
      // ... ...
      if (registry == null){
        registry = new MetricRegistry();
      }
      if (properties == null){
        properties = new HashMap<String,String>();
      }
      if (listener == null){
        listener((a,b) -> {});
      }
      if (gossipMembers == null) {
        gossipMembers = new ArrayList<>();
      }

      // **************************************************************
      // 创建消息处理器.
      // **************************************************************
      if (messageHandler == null) {
        messageHandler = MessageHandlerFactory.defaultHandler();
      }
	  
      return new GossipManager(cluster, uri, id, properties, settings, gossipMembers, listener, registry, messageHandler) {} ;
} // end 
```
### (4). MessageHandlerFactory.defaultHandler
```
public class MessageHandlerFactory {

  public static MessageHandler defaultHandler() {
  	// ***********************************************************  
	// 1. TypedMessageHandler实现了:MessageHandler
	// 2. 通过TypedMessageHandler包裹着:数据类型与MessageHandler的关系,这里用到了设计模式中的:中介者模式
	// 3. 通过concurrentHandler方法,创建一个临时:MessageHandler,内部持着List<TypedMessageHandler>,会挨个遍历这个集合,判断数据类型是否支持,如果支持,则委派给相应的MessageHandler
	// ***********************************************************  
    return concurrentHandler(
        new TypedMessageHandler(Response.class, new ResponseHandler()),
        new TypedMessageHandler(ShutdownMessage.class, new ShutdownMessageHandler()),
        new TypedMessageHandler(PerNodeDataMessage.class, new PerNodeDataMessageHandler()),
        new TypedMessageHandler(SharedDataMessage.class, new SharedDataMessageHandler()),
        new TypedMessageHandler(ActiveGossipMessage.class, new ActiveGossipMessageHandler()),
        new TypedMessageHandler(PerNodeDataBulkMessage.class, new PerNodeDataBulkMessageHandler()),
        new TypedMessageHandler(SharedDataBulkMessage.class, new SharedDataBulkMessageHandler())
    );
  } // end 
  
  
  public static MessageHandler concurrentHandler(MessageHandler... handlers) {
    if (handlers == null)
      throw new NullPointerException("handlers cannot be null");
    if (Arrays.asList(handlers).stream().filter(i -> i != null).count() != handlers.length) {
      throw new NullPointerException("found at least one null handler");
    }
  	
  	// ***********************************************************  
  	// 4. 通过创建匿名的MessageHandler,包裹着:List<TypedMessageHandler>
	//    当调用:invoke时,会挨个遍历所有的:TypedMessageHandler,而,TypedMessageHandler内部有持有真实的:MessageHandler
	//    这里用到了:设计模式中的:组合模式.
  	// ***********************************************************  
    return new MessageHandler() {
      @Override public boolean invoke(GossipCore gossipCore, GossipManager gossipManager,
              Base base) {
        return Arrays.asList(handlers).stream()
                .filter((mi) -> mi.invoke(gossipCore, gossipManager, base)).count() > 0;
      }
    };
  } // end 
  
}
```
### (5). TypedMessageHandler
> 其实,看完代码我觉得有点绕,为什么不通过一个Map来存储数据类型与MessageHandler的关系,直接根据数据类型定位到:MessageHandler即可,而不需要遍历. 


```
public class TypedMessageHandler implements MessageHandler {
  final private Class<?> messageClass;
  // 比如: ActiveGossipMessageHandler
  final private MessageHandler messageHandler;

  public TypedMessageHandler(Class<?> messageClass, MessageHandler messageHandler) {
    if (messageClass == null || messageHandler == null) {
      throw new NullPointerException();
    }
    this.messageClass = messageClass;
    this.messageHandler = messageHandler;
  }

  @Override
  public boolean invoke(GossipCore gossipCore, GossipManager gossipManager, Base base) {
    // 验证Base是否支持的类型
    if (messageClass.isAssignableFrom(base.getClass())) {
      // ******************************************************************
	  // 转交给真实的MessageHandler处理.
	  // ******************************************************************
      messageHandler.invoke(gossipCore, gossipManager, base);
      return true;
    } else {
      return false;
    }
  }
}
```
### (6). 看下MessageHandler接口能力
```
public interface MessageHandler {
  boolean invoke(GossipCore gossipCore, GossipManager gossipManager, Base base);
}
```
### (7). 设计模式
先不说代码咋样,就这一点点代码,你发现用到了如下设计模式:
+ 建造者模式
+ 工厂模式
+ 中介者模式
+ 组合模式 

### (8). 总结
从MessageHandler的接口能力上就能看出来,它大概就是接受报文请求,然后,根据不同听报文体,分发给相应的MessageHandler.  