---
layout: post
title: 'Gossip源码之UdpTransportManager发送成员信息(五)' 
date: 2022-11-07
author: 李新
tags:  Gossip
---

### (1). 概述
前面分析了UdpTransportManager的功能之一就是接受消息并处理,在这里,剖析它的另一大功能,就是发送成员信息.

### (2).  UdpTransportManager构造器
> 在前面对UdpTransportManager的构造器只是一带而过,实际:UdpTransportManager继承于:AbstractTransportManager

```
public UdpTransportManager(GossipManager gossipManager, GossipCore gossipCore) {
    super(gossipManager, gossipCore);
    // ... ...
}
```
### (3). AbstractTransportManager构造器
```
public AbstractTransportManager(GossipManager gossipManager, GossipCore gossipCore) {
    this.gossipManager = gossipManager;
    this.gossipCore = gossipCore;
    gossipThreadExecutor = Executors.newCachedThreadPool();
	// ********************************************************
	// 通过反射,创建:AbstractActiveGossiper的实现类
	// org.apache.gossip.manager.SimpleActiveGossiper
	// ********************************************************
    activeGossipThread = ReflectionUtils.constructWithReflection(
      gossipManager.getSettings().getActiveGossipClass(),
        new Class<?>[]{
            GossipManager.class, GossipCore.class, MetricRegistry.class
        },
        new Object[]{
            gossipManager, gossipCore, gossipManager.getRegistry()
        });
} // end 
```
### (4). AbstractTransportManager.startActiveGossiper
```
public void startActiveGossiper() {
	// ****************************************************************
	// activeGossipThread = org.apache.gossip.manager.SimpleActiveGossiper
	// ****************************************************************
	activeGossipThread.init();
}
```
### (5). SimpleActiveGossiper.init
```
public void init() {
    super.init();
    // *************************************************************************************
    // sendToALiveMember 发送在线成员列表
    // *************************************************************************************
    scheduledExecutorService.scheduleAtFixedRate(() -> {
      threadService.execute(() -> {
        sendToALiveMember();
      });
    }, 0, gossipManager.getSettings().getGossipInterval(), TimeUnit.MILLISECONDS);


    // *************************************************************************************
    // sendToDeadMember发送离线成员列表
	// 在前面有剖析过,其余的种子成员,默认状态是下线状态的(DOWN)
    // *************************************************************************************
    scheduledExecutorService.scheduleAtFixedRate(() -> {
      sendToDeadMember();
    }, 0, gossipManager.getSettings().getGossipInterval(), TimeUnit.MILLISECONDS);

    // ... ...
  }
```
### (6). SimpleActiveGossiper.sendToDeadMember
```
protected void sendToDeadMember(){
	// 通过随机函数,过滤出来一个:状态为下线(DOWN)成员
	LocalMember member = selectPartner(gossipManager.getDeadMembers());
	// 委托给sendMembershipList方法
	// 注意:mySelf是本机(me)成员,而,member是随机其它种子成员. 
	sendMembershipList(gossipManager.getMyself(), member);
}
```
### (7). SimpleActiveGossiper.sendMembershipList
```
protected void sendMembershipList(LocalMember me, LocalMember member) {
    if (member == null){
      return;
    }
	
    long startTime = System.currentTimeMillis();
    me.setHeartbeat(System.nanoTime());
	
	// ****************************************************************
	// 1. 创建报文(UdpActiveGossipMessage),内剖持有一个集合列表. 
	// ****************************************************************
    UdpActiveGossipMessage message = new UdpActiveGossipMessage();
    message.setUriFrom(gossipManager.getMyself().getUri().toASCIIString());
    message.setUuid(UUID.randomUUID().toString());
	// ****************************************************************
	// 2. 集合列表中的第一个成员是本机成员(me)
	// ****************************************************************
    message.getMembers().add(convert(me));
	
	// ****************************************************************
	// 3. 其余的成员,都往集合的第一个成员后添加,这也是能解释,为什么上一篇(接受消息)时,要标记出第一个成员,因为第一个成员发过来的消息,肯定是活着的. 
	// ****************************************************************
    for (LocalMember other : gossipManager.getMembers().keySet()) {
      message.getMembers().add(convert(other));
    }
	
	// ****************************************************************
	// 4. 委派给GossipCore.send方法进行消息发送,这个没有什么好去剖析的了,肯定是委托给:TransportManager做处理. 
	// ****************************************************************
    Response r = gossipCore.send(message, member.getUri());
    if (r instanceof ActiveGossipOk){
    } else {
      LOGGER.debug("Message " + message + " generated response " + r);
    }
    sendMembershipHistogram.update(System.currentTimeMillis() - startTime);
} //end 
```
### (8). 总结
UdpTransportManager的startActiveGossiper方法,会以固定的时间,向其它成员发送心跳信息,至此,Gossip大体的流程剖析完毕了.  