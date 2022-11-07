---
layout: post
title: 'Gossip源码剖析之RandomActiveGossipThread(三)' 
date: 2022-11-07
author: 李新
tags:  Gossip
---


### (1). 概述

### (2). 先看下RandomActiveGossipThread类的继承关系
```
com.google.code.gossip.manager.ActiveGossipThread  
  com.google.code.gossip.manager.impl.SendMembersActiveGossipThread
    com.google.code.gossip.manager.random.RandomActiveGossipThread
```
### (3). RandomActiveGossipThread
> RandomActiveGossipThread主要作用是从成员列表中,随机选择一个成员. 

```
public class RandomActiveGossipThread extends SendMembersActiveGossipThread {
  private final Random random;
	
  public RandomActiveGossipThread(GossipManager gossipManager) {
	super(gossipManager);
	random = new Random();
  } // end 构建器
  
  // **************************************************************************
  // 从成员列表中,随机选择一个成员.
  // **************************************************************************
  protected LocalGossipMember selectPartner(List<LocalGossipMember> memberList) {
	  LocalGossipMember member = null;
	  if (memberList.size() > 0) {
		int randomNeighborIndex = random.nextInt(memberList.size());
		member = memberList.get(randomNeighborIndex);
	  } else {
		GossipService.LOGGER.debug("I am alone in this world.");
	  }
	  return member;
  }// end   
}	
```
### (4). SendMembersActiveGossipThread
```
abstract public class SendMembersActiveGossipThread extends ActiveGossipThread {
	protected ObjectMapper om = new ObjectMapper();
   
	public SendMembersActiveGossipThread(GossipManager gossipManager) {
		super(gossipManager);
	} // end SendMembersActiveGossipThread
	
	
	// ***********************************************************************************
	// me          : 当前成员
	// memberList  : 其它成员
	// ***********************************************************************************
	protected void sendMembershipList(LocalGossipMember me, List<LocalGossipMember> memberList) {
	    GossipService.LOGGER.debug("Send sendMembershipList() is called.");
	    me.setHeartbeat(System.currentTimeMillis());
		
		// ***********************************************************************************
		// 1. 从其它成员列表中,随机选择一个成员
		// ***********************************************************************************
	    LocalGossipMember member = selectPartner(memberList);
	    if (member == null) {
	      return;
	    }
		
		// ***********************************************************************************
		// 2.创建Socket(UDP)
		// ***********************************************************************************
	    try (DatagramSocket socket = new DatagramSocket()) {
	      socket.setSoTimeout(gossipManager.getSettings().getGossipInterval());
		  
		  // ***********************************************************************************
		  // 3. 目标地址: 127.0.0.1:50002 
		  // ***********************************************************************************
	      InetAddress dest = InetAddress.getByName(member.getHost());
		  
		  // ***********************************************************************************
		  // 4. 把当前成员和其它所有成员都添加到ActiveGossipMessage集合里
		  //    ActiveGossipMessage相当于是一个集合.
		  // ***********************************************************************************
	      ActiveGossipMessage message = new ActiveGossipMessage();
		  // 第一个成员是本地成员,余下的成员是其它成员列表
	      message.getMembers().add(convert(me));
	      for (LocalGossipMember other : memberList) {
	        message.getMembers().add(convert(other));
	      }
		  
		  // ***********************************************************************************
		  // 5. 通过Jackson把ActiveGossipMessage进行序列化
		  // ***********************************************************************************
	      byte[] json_bytes = om.writeValueAsString(message).getBytes();
	      int packet_length = json_bytes.length;
	      if (packet_length < GossipManager.MAX_PACKET_SIZE) {
			// ***********************************************************************************
			// 6. 
			// ***********************************************************************************
	        byte[] buf = createBuffer(packet_length, json_bytes);
			// 通过Socket(UDP)发送消息: 127.0.0.1:50002 
	        DatagramPacket datagramPacket = new DatagramPacket(buf, buf.length, dest, member.getPort());
	        socket.send(datagramPacket);
	      } else {
	        GossipService.LOGGER.error("The length of the to be send message is too large (" + packet_length + " > " + GossipManager.MAX_PACKET_SIZE + ").");
	      }
	    } catch (IOException e1) {
	      GossipService.LOGGER.warn(e1);
	    }
	} // end 
	
	// ***********************************************************************************
	// 4.1 把LocalGossipMember转换成:GossipMember
	// ***********************************************************************************
	private GossipMember convert(LocalGossipMember member){
	    GossipMember gm = new GossipMember();
	    gm.setCluster(member.getClusterName());
	    gm.setHeartbeat(member.getHeartbeat());
	    gm.setHost(member.getHost());
	    gm.setId(member.getId());
	    gm.setPort(member.getPort());
	    return gm;
	} // end 
	
	
	// ***********************************************************************************
	// 6.1 通过ByteBuffer把消息体进行包裹.
	// ***********************************************************************************
	private byte[] createBuffer(int packetLength, byte[] jsonBytes) {
	    byte[] lengthBytes = new byte[4];
	    lengthBytes[0] = (byte) (packetLength >> 24);
	    lengthBytes[1] = (byte) ((packetLength << 8) >> 24);
	    lengthBytes[2] = (byte) ((packetLength << 16) >> 24);
	    lengthBytes[3] = (byte) ((packetLength << 24) >> 24);
	    ByteBuffer byteBuffer = ByteBuffer.allocate(4 + jsonBytes.length);
	    byteBuffer.put(lengthBytes);
	    byteBuffer.put(jsonBytes);
	    byte[] buf = byteBuffer.array();
	    return buf;
	} // end 
	
}
```
### (5). ActiveGossipThread
```
abstract public class ActiveGossipThread implements Runnable {
	
	protected final GossipManager gossipManager;
	
	private final AtomicBoolean keepRunning;
	
	public ActiveGossipThread(GossipManager gossipManager) {
		this.gossipManager = gossipManager;
	    this.keepRunning = new AtomicBoolean(true);
	} //end 构建器
	
	@Override
	public void run() {
		while (keepRunning.get()) {
	      try {
			// ***************************************************************** 
			// 休眠一段时间,然后,随机向其它成员发送消息.
			// ***************************************************************** 
	        TimeUnit.MILLISECONDS.sleep(gossipManager.getSettings().getGossipInterval());
	        sendMembershipList(gossipManager.getMyself(), gossipManager.getMemberList());
	      } catch (InterruptedException e) {
	        GossipService.LOGGER.error(e);
	        keepRunning.set(false);
	      }
	    } // end while
	    shutdown();
	}
	
	
}
```
### (6). 总结
RandomActiveGossipThread的主要作用是随机向其它成员发送消息.