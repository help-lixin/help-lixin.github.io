---
layout: post
title: 'Gossip源码之UdpTransportManager接受消息并处理(四)' 
date: 2022-11-07
author: 李新
tags:  Gossip
---

### (1). 概述
前面剖析到:GossipManager内部会创建UdpTransportManager进行消息管理,在这一小篇,主要剖析接收消息后,如何进行处理的. 

### (2). UdpTransportManager.startEndpoint
```
public class UdpTransportManager 
       extends AbstractTransportManager 
	   // ***********************************************************************
	   // 1. 它实现Runnable
	   // ***********************************************************************
	   implements Runnable {
	
	private final Thread me;
	
	public UdpTransportManager(GossipManager gossipManager, GossipCore gossipCore) {
	    super(gossipManager, gossipCore);
	    soTimeout = gossipManager.getSettings().getGossipInterval() * 2;
	    try {
	      SocketAddress socketAddress = new InetSocketAddress(gossipManager.getMyself().getUri().getHost() , gossipManager.getMyself().getUri().getPort());
		  // udp://localhost:10000
	      server = new DatagramSocket(socketAddress);
	    } catch (SocketException ex) {
	      LOGGER.warn(ex);
	      throw new RuntimeException(ex);
	    }
		
		// ***********************************************************************
		// 2. 创建了一个线程,指定Runnable为this,所以,这个类,要是Runnable的实现类,并且,要求要实现:run方法.
		// ***********************************************************************
	    me = new Thread(this);
	} // end 
	
	
	// ***********************************************************************
	// 3. 直接Thread.start方法
	// ***********************************************************************
	public void startEndpoint() {
	    me.start();
    } // end 
}	
```
### (3). UdpTransportManager.run
```
public void run() {
    while (keepRunning.get()) {
      try {
        // **********************************************************
        // 1. 通过UDP读取消息.
        // **********************************************************
        byte[] buf = read();
        try {
           // 通过协议管理器,进行解码
          Base message = gossipManager.getProtocolManager().read(buf);
		  // ********************************************************
		  // 2. 委派给GossipCore.receive方法,进行消息的处理.
		  // ********************************************************
          // 委派给GossipCore进行处理
          gossipCore.receive(message);
          //TODO this is suspect
          gossipManager.getMemberStateRefresher().run();
        } catch (RuntimeException ex) {//TODO trap json exception
          LOGGER.error("Unable to process message", ex);
        }
      } catch (IOException e) {
        if (!(e.getCause() instanceof InterruptedException)) {
          LOGGER.error(e);
        }
        keepRunning.set(false);
      }
    }
} // end 
```
### (4). GossipCore.receive
```
 public void receive(Base base) {
	 // ************************************************************
	 // 前面有剖析过,会遍历MessageHandler(TypedMessageHandler),判断数据类型(Base)是否支持,如果支持,则invoke,并返回:true
	 // ************************************************************
    if (!gossipManager.getMessageHandler().invoke(this, gossipManager, base)) {
      // 当没有找到相应的:MessageHandler时,提示处理不存在
      LOGGER.warn("received message can not be handled");
    }
} // end 
```
### (5). ActiveGossipMessageHandler
> 在这里以:ActiveGossipMessageHandler为例

```
public class ActiveGossipMessageHandler implements MessageHandler {
    
	@Override
	  public boolean invoke(GossipCore gossipCore, GossipManager gossipManager, Base base) {
	    List<Member> remoteGossipMembers = new ArrayList<>();
	    RemoteMember senderMember = null;
	    UdpActiveGossipMessage activeGossipMessage = (UdpActiveGossipMessage) base;
	    // *************************************************************************
		// 遍历所有的成员列表
		// *************************************************************************
	    for (int i = 0; i < activeGossipMessage.getMembers().size(); i++) {
	      URI u;
	      try {
	        u = new URI(activeGossipMessage.getMembers().get(i).getUri());
	      } catch (URISyntaxException e) {
	        GossipCore.LOGGER.debug("Gossip message with faulty URI", e);
	        continue;
	      }
	      RemoteMember member = new RemoteMember(
	              activeGossipMessage.getMembers().get(i).getCluster(),
	              u,
	              activeGossipMessage.getMembers().get(i).getId(),
	              activeGossipMessage.getMembers().get(i).getHeartbeat(),
	              activeGossipMessage.getMembers().get(i).getProperties());
	      // 标记第一个为发送者成员
	      if (i == 0) {
	        senderMember = member;
	      }
	
	      // 处理集群名称相同的成员
	      if (!(member.getClusterName().equals(gossipManager.getMyself().getClusterName()))) {
	        UdpNotAMemberFault f = new UdpNotAMemberFault();
	        f.setException("Not a member of this cluster " + i);
	        f.setUriFrom(activeGossipMessage.getUriFrom());
	        f.setUuid(activeGossipMessage.getUuid());
	        GossipCore.LOGGER.warn(f);
	        gossipCore.sendOneWay(f, member.getUri());
	        continue;
	      }
	      remoteGossipMembers.add(member);
	    }
	
	    // apache gossip好像对Google进行了增强,最终会发送一个请求,告之对方.
	    UdpActiveGossipOk o = new UdpActiveGossipOk();
	    o.setUriFrom(activeGossipMessage.getUriFrom());
	    o.setUuid(activeGossipMessage.getUuid());
	    gossipCore.sendOneWay(o, senderMember.getUri());
	
	    // *************************************************
	    // 最终会处理member信息.
	    // *************************************************
	    gossipCore.mergeLists(senderMember, remoteGossipMembers);
	    return true;
	  }
	
}
```
### (6). GossipCore.mergeLists
```
public void mergeLists(RemoteMember senderMember, List<Member> remoteList) {
    // senderMember 代表着远程发送消息过来的程序
	// 1. 本进程记录的:下线的成员列表中,如果,包含有:发送消息过来的成员,则先标记这个成员的心跳时间.
    for (LocalMember i : gossipManager.getDeadMembers()) {
      if (i.getId().equals(senderMember.getId())) {
        LOGGER.debug(gossipManager.getMyself() + " contacted by dead member " + senderMember.getUri());
        i.recordHeartbeat(senderMember.getHeartbeat());
        i.setHeartbeat(senderMember.getHeartbeat());
        //TODO consider forcing an UP here
      }
    }

    // 2. 遍历所有的成员列表
    for (Member remoteMember : remoteList) {
	  // 如果,遍历的某一个成员信息与本地(me)成员相同,则跳过.	
      if (remoteMember.getId().equals(gossipManager.getMyself().getId())) {
        continue;
      }
	  
	  // 3. 把RemoteMember转换成:LocalMember
      LocalMember aNewMember = new LocalMember(remoteMember.getClusterName(),
      remoteMember.getUri(),
      remoteMember.getId(),
      remoteMember.getHeartbeat(),
      remoteMember.getProperties(),
      gossipManager.getSettings().getWindowSize(),
      gossipManager.getSettings().getMinimumSamples(),
      gossipManager.getSettings().getDistribution());
      aNewMember.recordHeartbeat(remoteMember.getHeartbeat());
      // ***************************************************************************
      // 4. 添加新的成员到:ConcurrentSkipListMap<LocalMember, GossipState>集合里,并标记为:UP
      // ***************************************************************************
      Object result = gossipManager.getMembers().putIfAbsent(aNewMember, GossipState.UP);
	  // 5. 当ConcurrentSkipListMap<LocalMember, GossipState>集合里,存在这个成员的情况下,重新标记这个成员的相关信息(Heartbeat)
      if (result != null){
        for (Entry<LocalMember, GossipState> localMember : gossipManager.getMembers().entrySet()){
          if (localMember.getKey().getId().equals(remoteMember.getId())){
            localMember.getKey().recordHeartbeat(remoteMember.getHeartbeat());
            localMember.getKey().setHeartbeat(remoteMember.getHeartbeat());
            localMember.getKey().setProperties(remoteMember.getProperties());
          }
        }
      } // end result
    }// end for
} // end 
```
### (7). 总结
UdpTransportManager的职责之一就是接受网络请求,并,转发请求给相应的:MessageHandler(比如:ActiveGossipMessageHandler),反正,最终MessageHandler会操纵GossipManager更新成员列表(ConcurrentSkipListMap<LocalMember, GossipState>)信息,注意一点就是:本机成员(me)是不会在这个集合里的. 