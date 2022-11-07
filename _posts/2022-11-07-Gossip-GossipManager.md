---
layout: post
title: 'Gossip源码剖析之GossipManager(二)' 
date: 2022-11-07
author: 李新
tags:  Gossip
---

### (1). 概述

### (2). 先看下RandomGossipManager继承关系
```
com.google.code.gossip.manager.GossipManager
    com.google.code.gossip.manager.random.RandomGossipManager
```
### (3). RandomGossipManager
```
public class RandomGossipManager extends GossipManager {
	
  public RandomGossipManager(String cluster, String address, int port, String id,
          GossipSettings settings, List<GossipMember> gossipMembers, GossipListener listener) {
	// ***************************************************************		  
	// OnlyProcessReceivedPassiveGossipThread:被动接受消息. 
	// RandomActiveGossipThread: 随机向其它成员发送消息.
	// ***************************************************************		  
    super( OnlyProcessReceivedPassiveGossipThread.class, 
	       RandomActiveGossipThread.class, cluster,
		   // 127.0.0.1
           address, 
		   // 50001
		   port, 
		   // "1"
		   id, 
		   // gossipInterval=1000,cleanupInterval=1000
		   settings, 
		   // 所有的成员列表
		   // Member [address=127.0.0.1:50001, id=1, heartbeat=1667828379424]
		   // Member [address=127.0.0.1:50002, id=2, heartbeat=1667828379424]
		   // Member [address=127.0.0.1:50003, id=3, heartbeat=1667828379424]
		   gossipMembers, 
		   // 监听器
		   listener);
  }
}
```
### (4). GossipManager构建器
```
// **********************************************************
// 继承于Thread,所以,重写run方法很重要
// **********************************************************
public abstract class GossipManager extends Thread implements NotificationListener {
  
  // 当前成员节点
  private final LocalGossipMember me;
  
  
  public GossipManager(Class<? extends PassiveGossipThread> passiveGossipThreadClass,
            Class<? extends ActiveGossipThread> activeGossipThreadClass, String cluster,
            String address, int port, String id, GossipSettings settings,
            List<GossipMember> gossipMembers, GossipListener listener) {
	  // OnlyProcessReceivedPassiveGossipThread
      this.passiveGossipThreadClass = passiveGossipThreadClass;
	  // RandomActiveGossipThread
      this.activeGossipThreadClass = activeGossipThreadClass;
      this.settings = settings;
	  
	  // 127.0.0.1:50001 
      me = new LocalGossipMember(cluster, address, port, id, System.currentTimeMillis(), this, settings.getCleanupInterval());
			  
      members = new ConcurrentSkipListMap<>();
	  
      for (GossipMember startupMember : gossipMembers) {
		// 排除当前成员节点:127.0.0.1:50001 
		// **********************members**********************
		// members为其它节点,内容如下:
		// 127.0.0.1:50002
		// 127.0.0.1:50003
        if (!startupMember.equals(me)) { 
          LocalGossipMember member = new LocalGossipMember(startupMember.getClusterName(),
                  startupMember.getHost(), startupMember.getPort(), startupMember.getId(),
                  System.currentTimeMillis(), this, settings.getCleanupInterval());
          members.put(member, GossipState.UP);
          GossipService.LOGGER.debug(member);
        }
      }
	  
	  // 创建线程池.
      gossipThreadExecutor = Executors.newCachedThreadPool();
      gossipServiceRunning = new AtomicBoolean(true);
      this.listener = listener;
      Runtime.getRuntime().addShutdownHook(new Thread(new Runnable() {
        public void run() {
          GossipService.LOGGER.debug("Service has been shutdown...");
        }
      }));
    } // end 构建器
	
	
	public LocalGossipMember getMyself() {
	    return me;
    } //end getMyself
}
```
### (5). GossipManager.run
```
public void run() {
    // 遍历所有的成员列表.
	for (LocalGossipMember member : members.keySet()) {
      if (member != me) {
        // 启动定时任务
        member.startTimeoutTimer();
      }
    }
    try {
      // ******************************************************************
	  // 通过反射创建:OnlyProcessReceivedPassiveGossipThread.
	  // OnlyProcessReceivedPassiveGossipThread主要用于被动接受请求.
	  // ******************************************************************
      passiveGossipThread = passiveGossipThreadClass.getConstructor(GossipManager.class).newInstance(this);
      gossipThreadExecutor.execute(passiveGossipThread);
	  
	  // ******************************************************************
	  // 通过反射创建:RandomActiveGossipThread
	  // RandomActiveGossipThread的主要用于随机向其它成员发送消息.  
	  // ******************************************************************
      activeGossipThread = activeGossipThreadClass.getConstructor(GossipManager.class).newInstance(this);
      gossipThreadExecutor.execute(activeGossipThread);
    } catch (InstantiationException | IllegalAccessException | IllegalArgumentException | InvocationTargetException | NoSuchMethodException | SecurityException e1) {
      throw new RuntimeException(e1);
    }
	
    GossipService.LOGGER.debug("The GossipService is started.");
    while (gossipServiceRunning.get()) {
      try {
        // TODO
        TimeUnit.MILLISECONDS.sleep(1);
      } catch (InterruptedException e) {
        GossipService.LOGGER.warn("The GossipClient was interrupted.");
      }
    } // end while
} // end 
```
### (6). 总结
GossipManager的主要作用于Hold住两个线程,这两个线程用于被动接受消息和随机向成员发送消息.  